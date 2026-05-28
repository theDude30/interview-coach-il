# interview-coach-il — Design Spec

**Date:** 2026-05-28
**Author:** Tzahi Bergman (with Claude)
**Status:** Draft for review

## 1. Purpose & Scope

A Hebrew-first web app that helps software engineers prepare for technical interviews. Phase 1 is a shareable, no-auth tool aimed at the author and a small group of friends. The app provides four modes built on a shared infrastructure of resume parsing, an LLM reasoning core, and Hebrew speech I/O:

1. **Resume ↔ JD Gap Analyzer** — upload resume + paste job descriptions, get a written strengths/gaps/study-plan report per target role.
2. **Text Mock Interview** — agent plays an interviewer, runs a multi-round mock based on a hybrid plan (agent suggests, user edits) tailored to the target role.
3. **Voice Mock Interview (Hebrew)** — same as the text mode, voice-driven via Whisper STT and OpenAI TTS.
4. **Hebrew Q&A Tutor** — free-form chat for technical study (algorithms, system design, behavioral framing).

**Non-goals for Phase 1:** real accounts, payments, multi-region deploy, mobile apps, real-time barge-in for voice, vector search.

## 2. Constraints & Decisions

| Decision | Value |
|---|---|
| Hosting model | Web app, single-region |
| Auth (Phase 1) | None — anonymous session tokens |
| LLM | Anthropic Claude (Sonnet/Opus) |
| STT | OpenAI Whisper (`language=he`) |
| TTS | OpenAI TTS (`tts-1`, voice configurable) |
| Backend | Node.js + Express + TypeScript |
| Frontend | Vite + React + TypeScript |
| DB | Managed Postgres on Fly.io |
| Audio storage (Phase 1) | Local disk under `/data/audio` on Fly volume |
| Data TTL | 7 days for everything (audio + text); session activity bumps it |
| Audio cap | 5 GB global, oldest-first eviction |
| Rate limit | 30 LLM calls/hr, 5 audio uploads/min, per session token |
| Cost meter | Visible in UI, server-computed |
| Mic UX | Toggle (click to start, click to stop). Spacebar shortcut. |
| Deploy target | **Fly.io** (Frankfurt region) |
| Key model | Operator-provided keys (Tzahi pays); no BYOK in Phase 1 |
| Repo | Monorepo, npm workspaces |

## 3. High-Level Architecture

### 3.1 Repo Layout

```
interview-coach-il/
├── apps/
│   ├── web/                # Vite + React + TS
│   └── api/                # Express + TS
├── packages/
│   └── shared/             # Shared types (DTOs, enums, Zod schemas)
├── package.json            # npm workspaces
├── docker-compose.yml      # local Postgres
└── README.md
```

### 3.2 Runtime Topology

```
Browser (React) ─► Express API ─► Anthropic Claude
                         │
                         ├──────► OpenAI Whisper (STT)
                         │
                         ├──────► OpenAI TTS
                         │
                         └──────► Postgres
                                  + /data/audio (local disk, GC'd hourly)
```

All third-party API keys live on the server. The browser never sees a vendor key. CORS is allowlisted to the production origin plus `localhost:5173`.

## 4. Local-First Development Model

Local containerized development is a first-class concern. The full stack — backend, frontend, Postgres, persistent audio volume — must run on the developer's laptop via a single command, with hot reload, before any cloud deploy happens. Production on Fly is a deployment target, not a development environment.

### 4.1 docker-compose.yml — the local stack

Four services at the repo root:

```yaml
# docker-compose.yml (Phase 1 sketch — concrete file written during implementation)
services:
  postgres:
    image: postgres:16-alpine
    environment:
      POSTGRES_DB: coach
      POSTGRES_USER: coach
      POSTGRES_PASSWORD: coach
    ports: ["5432:5432"]
    volumes: [pg-data:/var/lib/postgresql/data]
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U coach"]
      interval: 5s

  mock-llm:
    build: ./apps/mock-llm
    ports: ["3100:3100"]
    profiles: ["mock"]   # only starts when --profile mock is passed

  api:
    build:
      context: .
      dockerfile: apps/api/Dockerfile.dev
    environment:
      DATABASE_URL: postgres://coach:coach@postgres:5432/coach
      AUDIO_TTL_DAYS: 7
      AUDIO_MAX_BYTES: 5368709120
      # ANTHROPIC_API_KEY / OPENAI_API_KEY come from .env, OR MOCK_LLM=1
    env_file: [.env]
    volumes:
      - ./apps/api/src:/app/apps/api/src
      - ./packages/shared:/app/packages/shared
      - audio-data:/data/audio
    ports: ["3000:3000"]
    depends_on:
      postgres: { condition: service_healthy }

  web:
    build:
      context: .
      dockerfile: apps/web/Dockerfile.dev
    volumes:
      - ./apps/web/src:/app/apps/web/src
      - ./packages/shared:/app/packages/shared
    ports: ["5173:5173"]
    depends_on: [api]

volumes:
  pg-data:
  audio-data:
```

### 4.2 Dockerfiles

Each app has two Dockerfiles:

- `apps/api/Dockerfile.dev` — Node 20 base, installs all deps (including dev), runs `tsx watch src/index.ts`. No source baked in; mounted at runtime for hot reload.
- `apps/api/Dockerfile` (prod) — multi-stage build: builder stage compiles TS, runtime stage is `node:20-alpine` with only `dist/` and prod deps. Used by Fly.
- `apps/web/Dockerfile.dev` — runs `vite dev --host 0.0.0.0`. HMR over the host-mapped port.
- `apps/web/Dockerfile` (prod) — runs `vite build`; the resulting `dist/` is **not** served by this image — instead it's copied into the api image's prod build, so production is a single container serving both API and static assets.

### 4.3 Mock LLM service

`apps/mock-llm/` is a small Express service that mimics Anthropic, OpenAI Whisper, and OpenAI TTS endpoints. Returns canned responses keyed by request fingerprint. Same fakes used by integration tests, so dev and test stay consistent.

When `MOCK_LLM=1` is set on the `api` service, the LLM/STT/TTS client wrappers (Section 11) route to `http://mock-llm:3100` instead of the real vendor endpoints. **Zero API spend during day-to-day UI work.**

Started via: `docker compose --profile mock up`. Default `docker compose up` does **not** start it — that requires real keys.

### 4.4 Developer workflows

**First-time setup:**

```bash
cp .env.example .env
# Option A: edit .env with real ANTHROPIC_API_KEY and OPENAI_API_KEY
# Option B: set MOCK_LLM=1 in .env and skip the keys entirely
docker compose up
# Open http://localhost:5173
```

**With mock LLM (no API spend):**

```bash
MOCK_LLM=1 docker compose --profile mock up
```

**Running tests:**

```bash
docker compose run --rm api npm test
```

Integration tests use testcontainers to spin up an ephemeral Postgres separate from the long-running `postgres` service — tests never touch dev data.

**Reset to clean state:**

```bash
docker compose down -v   # also wipes pg-data and audio-data
docker compose up
```

**Convenience scripts** (root `package.json`):

- `npm run dev` → `docker compose up`
- `npm run dev:mock` → `MOCK_LLM=1 docker compose --profile mock up`
- `npm run dev:reset` → `docker compose down -v && docker compose up`
- `npm run prod:local` → see §4.5

### 4.5 Production parity check

A second compose file, `docker-compose.prod-local.yml`, runs the **production** Dockerfiles against local Postgres. No hot reload, no source mounts — the actual production image as it would ship to Fly. Used to smoke-test the prod image before `fly deploy`.

```bash
docker compose -f docker-compose.yml -f docker-compose.prod-local.yml up --build
```

This catches issues that hot-reload dev mode hides:

- Missing files in the prod `Dockerfile` `COPY` list.
- Build-time-only deps accidentally needed at runtime.
- Static asset paths that work via Vite dev server but break when served by Express.
- Environment variable handling in the production entrypoint.

The rule: if `npm run prod:local` doesn't work, `fly deploy` won't either. Run it before every deploy in Phase 1.

### 4.6 Local-to-production parity table

| Concern | Local (`docker compose up`) | Production (Fly.io) |
|---|---|---|
| Backend container | `api` service, `Dockerfile.dev` (hot reload) | `api` machine, `Dockerfile` (prod build) |
| Frontend | Vite dev server, separate container, HMR | Static assets baked into the api prod image |
| Database | `postgres:16-alpine` in compose, named volume | Fly Managed Postgres 16, automatic snapshots |
| Audio storage | Named docker volume at `/data/audio` | Fly persistent volume at `/data` |
| API keys | `.env` file (gitignored) loaded via `env_file` | `fly secrets set` injects as env vars |
| `MOCK_LLM` flag | Available via `--profile mock` | Available — useful for staging without spend |
| TLS | None (`http://localhost:5173`) | Let's Encrypt, auto-provisioned |
| Postgres backup | None (`docker compose down -v` to reset) | Daily snapshots managed by Fly |

The parity is deliberate: anything that runs cleanly under `npm run prod:local` will run cleanly on Fly.

## 5. Session Model

The app is "anonymous but persistent." On first visit, the backend issues a session token (22-char base62, ≥130 bits of entropy from CSPRNG). The token is stored in browser localStorage and sent as `X-Session-Token` on every API call. The URL form `?s=<token>` re-attaches a session — this is how a user shares progress across devices or with a friend.

**Implication:** the token is a bearer credential. Anyone with it has full access to that session's data. Acceptable for Phase 1, replaced by real auth in Phase 2.

**Session lifecycle:**
- `POST /sessions` issues a fresh token.
- Every authenticated request bumps `sessions.last_active_at`.
- The hourly GC deletes sessions inactive for more than `SESSION_TTL_DAYS` (default 7), cascading to all dependent rows and audio files.
- A user returning after expiry lands on an empty session and sees a banner: "This session expired; starting fresh."

## 6. Data Model (Postgres)

```sql
sessions
  id              text primary key            -- the token
  created_at      timestamptz not null
  last_active_at  timestamptz not null
  display_name    text

resumes
  id              uuid primary key
  session_id      text references sessions(id) on delete cascade
  filename        text
  raw_text        text not null
  parsed_json     jsonb not null              -- skills, roles, years, etc.
  created_at      timestamptz not null

target_roles
  id              uuid primary key
  session_id      text references sessions(id) on delete cascade
  company         text not null
  role_title      text not null
  jd_text         text not null
  created_at      timestamptz not null

interview_plans
  id              uuid primary key
  session_id      text references sessions(id) on delete cascade
  target_role_id  uuid references target_roles(id) on delete set null    -- preserve historical plans
  rounds          jsonb not null              -- [{type, focus, duration_min, difficulty, sample_topics[]}]
  mode            text not null               -- 'text' | 'voice'
  hint_level      text not null               -- 'none' | 'subtle' | 'explicit'
  persona         text not null               -- 'friendly' | 'neutral' | 'challenging'
  language        text not null               -- 'he' | 'en'
  status          text not null               -- 'draft' | 'active' | 'completed'
  created_at      timestamptz not null

interview_sessions
  id              uuid primary key
  plan_id         uuid references interview_plans(id) on delete cascade
  started_at      timestamptz not null
  ended_at        timestamptz
  feedback_json   jsonb                       -- final scoring/notes

messages
  id              uuid primary key
  interview_id    uuid references interview_sessions(id) on delete cascade
  role            text not null               -- 'interviewer' | 'candidate' | 'system'
  text            text not null
  audio_url       text                        -- nullable; cleared when GC'd
  round_index     int                         -- which round this message belongs to
  created_at      timestamptz not null

qa_threads
  id              uuid primary key
  session_id      text references sessions(id) on delete cascade
  title           text not null
  created_at      timestamptz not null

qa_messages
  id              uuid primary key
  thread_id       uuid references qa_threads(id) on delete cascade
  role            text not null               -- 'user' | 'assistant'
  text            text not null
  created_at      timestamptz not null

usage_log
  id              bigserial primary key
  session_id      text references sessions(id) on delete cascade
  service         text not null               -- 'claude' | 'whisper' | 'tts'
  operation       text not null               -- e.g. 'interview_turn', 'gap_analysis'
  tokens_in       int
  tokens_out      int
  cache_read_tokens   int
  cache_creation_tokens int
  audio_seconds   numeric
  cost_usd        numeric not null
  created_at      timestamptz not null

audit_log
  id              bigserial primary key
  session_id      text references sessions(id) on delete cascade
  event_type      text not null               -- 'refusal' | 'rate_limit' | 'parse_failure' | ...
  payload         jsonb
  created_at      timestamptz not null
```

Indexes: `sessions(last_active_at)` for GC; `messages(interview_id, created_at)`; `usage_log(session_id, created_at)`.

## 7. Retention & Garbage Collection

A single in-process hourly job (`node-cron`) handles all TTL work:

1. **Audio TTL.** Delete every file under `/data/audio` older than `AUDIO_TTL_DAYS` (default 7). For each deleted file, clear `messages.audio_url`.
2. **Audio cap.** If `/data/audio` total size exceeds `AUDIO_MAX_BYTES` (default 5 GB), delete oldest files first until under the cap.
3. **Session TTL.** Delete every `sessions` row where `last_active_at < now() - SESSION_TTL_DAYS`. Cascade deletes handle the rest of the data.

UX implications:
- A returning user whose audio expired but session is still active sees transcripts with a small "audio expired" note next to each turn. Replay works for text.
- A returning user whose session expired lands on a fresh session, sees a one-line banner.

## 8. The Four User Modes

### 7.1 Resume ↔ JD Gap Analyzer

1. User uploads resume (PDF/DOCX, ≤5 MB).
2. Backend extracts text (`pdf-parse` for PDF, `mammoth` for DOCX), then calls Claude with `resumeExtractor` prompt to produce structured JSON via tool-use.
3. User pastes one or more JDs, each tagged with company + role title.
4. User clicks "Analyze." Backend streams a per-role markdown report via SSE:
   - `## Strengths` — concrete matches against JD requirements.
   - `## Gaps` — missing skills/experience, ranked by JD emphasis.
   - `## Study Plan` — 2-week prioritized list.
   - `## Likely Interview Format` — uses a constrained vocabulary (e.g., `algo`, `system_design`, `behavioral`, `coding`, `take_home`) so the planner can consume it.
5. Output rendered with `react-markdown`, RTL-aware. User can export as `.md`.

### 7.2 Text Mock Interview

1. User selects a target role (saved or ad-hoc).
2. Backend calls `interviewPlanner` prompt with resume JSON + JD + likely-format hint. Returns structured `rounds[]` via tool-use.
3. User edits the plan in the UI (skip rounds, swap focus, adjust difficulty). Saves → `interview_plans.status = 'active'`.
4. User clicks "Start." Chat UI opens.
5. Each user turn POSTs to `/interview-sessions/:id/turn` (SSE response). Backend calls `interviewer` prompt with:
   - System: composed persona + language directive + round metadata + hint policy + behavioral rules + stop conditions.
   - User content: parsed resume JSON + plan + summarized prior context + last 20 verbatim turns.
   - Tools: `transition_round(summary)`, `end_interview()`.
6. When Claude calls `transition_round`, the server emits a `round_transition` SSE event, persists a `system` message with the round summary, advances `round_index`, and re-issues the system prompt with the new round metadata on the next turn.
7. When Claude calls `end_interview`, the server marks `interview_sessions.ended_at` and triggers `feedbackGenerator`.
8. Feedback (structured JSON: per-round score + rationale, top-3 strengths, top-3 improvements, quoted moments) is stored in `feedback_json` and shown in the UI.

### 7.3 Voice Mock Interview (Hebrew)

Same as 7.2, plus an audio loop per turn:

1. User clicks mic toggle (or taps spacebar). Browser captures via `MediaRecorder` (webm/opus, 16 kHz mono).
2. On stop, audio POSTs to `/interview-sessions/:id/voice-turn` (multipart, ≤25 MB).
3. Backend pipeline:
   - **Whisper** transcribes with `language=he`. A transcript is flagged as suspicious if any of: (a) length < 5 chars and audio > 2 s, (b) more than 30% non-Hebrew characters while `language=he` was set, (c) Whisper's `no_speech_prob` (when available via verbose response) exceeds 0.6. Suspicious transcripts are surfaced to the user with an "Edit before sending?" prompt instead of being auto-submitted.
   - Transcript appended as a `candidate` message.
   - **Claude** generates the interviewer reply, streaming.
   - As each sentence completes, it's sent to **OpenAI TTS** in parallel. Audio files are persisted to `/data/audio/<session_id>/<message_id>_<sentence_idx>.mp3`.
   - SSE channel emits `token` events (text) and `audio_chunk` events (URL + sentence index) interleaved.
4. Browser plays audio chunks sequentially while text appears in sync.
5. Mic is disabled during agent playback. A visual indicator shows mic state.
6. **Latency target:** time-to-first-audio < 2.5 s. If consistently missed in testing, add an "intro filler" played from a cached library while the real response generates.
7. Barge-in is **not** in v1.

### 7.4 Hebrew Q&A Tutor

1. Free-form chat. User asks any technical question (Hebrew or English).
2. Backend calls `tutor` prompt — teaching tone, code blocks, mermaid diagrams when useful, prefers Hebrew unless asked otherwise, asks clarifying questions on ambiguity.
3. Threads are persistent (within TTL), listed in the sidebar.
4. Voice input is available (same Whisper pipeline). Voice output is opt-in here — long technical answers read better than they listen.
5. **Note:** Q&A tutor does not use `interview_plans`. Language preference is global (the top-bar toggle); the `tutor` prompt picks it up from session state, not from a plan row.

### 7.5 Shared UI Pieces

- **Language toggle** (top bar) swaps UI strings and the LLM language directive. Tailwind RTL utilities flip layout.
- **Hint level slider** (Modes 2 & 3): `none` / `subtle` / `explicit`.
- **Cost meter** (top bar): `~$0.42`, tooltip breaks down by service.
- **Restart with fresh context** button in every mode.
- **Export** in every mode (markdown).

## 9. Backend API Surface

All routes under `/api/v1`. Session token via `X-Session-Token` header. Streaming endpoints use SSE.

```
POST   /sessions                              → { token, created_at }
GET    /sessions/me
PATCH  /sessions/me
DELETE /sessions/me

POST   /resumes                  (multipart)
GET    /resumes
GET    /resumes/:id
DELETE /resumes/:id

POST   /target-roles
GET    /target-roles
DELETE /target-roles/:id

POST   /gap-analysis             (SSE)
       body: { resume_id, target_role_ids: [...] }

POST   /interview-plans
PATCH  /interview-plans/:id
GET    /interview-plans/:id

POST   /interview-sessions
GET    /interview-sessions/:id
POST   /interview-sessions/:id/turn        (SSE; text mode)
POST   /interview-sessions/:id/voice-turn  (multipart in; SSE out for tokens + audio chunks)
POST   /interview-sessions/:id/end

POST   /qa-threads
GET    /qa-threads
POST   /qa-threads/:id/messages  (SSE)
GET    /qa-threads/:id

GET    /audio/:message_id.mp3              → 200 with audio, 404 if expired (no error)
GET    /usage/me                           → { total_usd, breakdown_by_service, since }
```

### 8.1 SSE Event Contract

```
event: token            data: {"text": "..."}
event: round_transition data: {"from_round": 1, "to_round": 2, "summary": "..."}
event: audio_chunk      data: {"url": "/audio/<id>.mp3", "sentence_index": 0}
event: cost_update      data: {"total_usd": 0.42, "delta_usd": 0.03}
event: done             data: {"message_id": "...", "total_tokens": 1240}
event: error            data: {"code": "...", "message": "...", "retry_after_ms": 5000}
```

Reconnect via `Last-Event-ID`.

### 8.2 Middleware Stack

In order:

1. `requestId` — UUID per request.
2. `cors` — allowlist for the production origin + `localhost:5173`.
3. `rateLimit` — token-bucket per session token. 30 LLM calls/hr, 5 audio uploads/min. Returns `429` with `Retry-After`. `POST /sessions` is exempt; everything else is gated.
4. `sessionAuth` — validates token, loads session, bumps `last_active_at`. `POST /sessions` skips this.
5. `requestLogger` — structured JSON (pino) with redaction of resume text and message bodies.
6. `errorHandler` — terminal; maps errors to `{ error: { code, message, request_id } }`.

## 10. Prompt Architecture

All prompts live in `apps/api/src/prompts/` as TypeScript modules exporting `{ version, render(input): string }`. **No inline prompts in route handlers.**

| Prompt | Inputs | Output |
|---|---|---|
| `resumeExtractor` | Raw resume text | Structured JSON via tool-use: skills[], roles[], years_total, education, languages, projects, summary |
| `gapAnalyzer` | Resume JSON, JD text, company | Streamed markdown with fixed section headings |
| `interviewPlanner` | Resume JSON, JD, likely-format hint | `rounds: [{type, focus, duration_min, difficulty, sample_topics[]}]` via tool-use |
| `interviewer` | Persona, language, round metadata, hint policy, summarized history, last 20 turns | Streamed reply; tool calls `transition_round` / `end_interview` |
| `feedbackGenerator` | Full transcript, plan, target role | Structured JSON via tool-use (scores, strengths, improvements, quoted moments) |
| `tutor` | Conversation history | Streamed teaching-tone reply, Hebrew by default |

Each prompt file exports a `version` string. Logs record `prompt_version` per call. Env var `PROMPT_OVERRIDE_<NAME>` can pin a specific version for rollbacks or A/B comparison.

### 9.1 Conversation Context Management

- Last 20 turns kept verbatim.
- Older turns summarized into a `prior_context` block, regenerated once per round transition.
- Resume JSON, plan, and current-round metadata re-injected every turn.
- Anthropic prompt caching (`cache_control: ephemeral`) applied to system prompt + resume JSON + plan, so the cached prefix is shared across all turns of an interview.

## 11. LLM/STT/TTS Client Wrappers

`apps/api/src/clients/claude.ts`:
- Wraps `@anthropic-ai/sdk`.
- Exponential backoff retry on 429 / 5xx (max 3 attempts).
- Logs every call to `usage_log` with token counts and computed USD cost (rates in `apps/api/src/config/pricing.ts`).
- Streaming abstraction yields normalized events.
- `MOCK_LLM=1` swaps to an in-memory fake returning canned responses keyed by prompt fingerprint.

`apps/api/src/clients/openai.ts`: same shape for Whisper and TTS.

## 12. Error Handling — User-Visible Contract

| Failure | User sees | Server does |
|---|---|---|
| Whisper times out | "Couldn't hear that, try again" + mic stays armed | log + metric `stt_timeout` |
| Claude rate limit | "Busy — retrying..." (auto-retry up to 3x) | exp backoff |
| Claude refuses (safety) | "Let's rephrase that question." | audit log entry |
| TTS fails | Text-only for that turn; mic stays usable | log |
| Network drop mid-stream | EventSource auto-reconnects; resumes from `Last-Event-ID` | server replays buffered events |
| Audio file expired | "Audio expired — transcript still available" | normal, not an error |
| Resume parse fails | "Couldn't parse that resume; paste as text instead?" | offers raw-text fallback |
| Rate limit hit | "You've hit your limit; try again in N min" | 429 with `Retry-After` |

## 13. Testing Strategy

**Unit (Vitest):** prompt template renderers (snapshot tests), pure helpers (conversation summarizer, cost calculator, rate limiter, resume parser post-processing), React component logic (RTL handling, hint slider, cost meter).

**Integration (Vitest + supertest):** every REST route end-to-end against a real Postgres via testcontainers. LLM/STT/TTS calls stubbed at the client boundary using the same fakes the `MOCK_LLM` flag enables. Critical paths: session lifecycle, resume upload → parse → fetch, gap analysis SSE stream completes, full text interview turn, full voice interview turn (audio fake returns fixed mp3), TTL GC actually deletes expired rows + files, rate limiter returns 429 after threshold.

**E2E (Playwright, gated):** three scenarios, runnable with `E2E=1` against real APIs. Not in default CI.
1. New session → upload resume → run gap analysis → see report.
2. New session → start text interview → complete one round → end → see feedback.
3. New session → start voice interview → speak one turn → hear reply.

**LLM evals (`apps/api/evals/`):** scripted harness runs the interviewer prompt against ~20 transcripts using Claude as judge. Scores behavioral consistency (one-question-at-a-time, language adherence, hint compliance). Results checked into the repo as JSON. On-demand, not on every commit.

## 14. Observability

- **Logs (pino):** `request_id`, `session_id`, `route`, `prompt_version`, `latency_ms`, token counts, cache hit counts. Resume text and message bodies redacted.
- **Metrics (`prom-client` at `/metrics`):** request counts, p50/p95/p99 per route, LLM error rates, GC throughput, `/data/audio` size.
- **`usage_log` table** drives the in-UI cost meter and ad hoc "which session is burning budget" queries.
- **`audit_log` table** for refusals, rate-limit hits, parse failures.
- **No external APM in Phase 1.** Platform's built-in log viewer is enough.

## 15. Deployment

- **Target:** Fly.io, Frankfurt region (`fra`) — closest to Israel (~70 ms RTT to Tel Aviv).
- **Backend instance:** `shared-cpu-1x`, 1 vCPU, 1 GB RAM, single machine in Phase 1. Scales horizontally later by lifting audio to S3 first.
- **Postgres:** Fly Managed Postgres, "Development" cluster (1 GB RAM, 10 GB disk, daily snapshots). Connection string injected via `DATABASE_URL` secret.
- **Volume:** Fly 10 GB persistent volume mounted at `/data`. Attached to the single machine. (When scaling out, audio moves to S3-compatible storage; see Section 19.)
- **TLS:** Let's Encrypt certificates auto-provisioned by Fly when the custom domain is added. Automatic renewal.
- **Domain:** registered separately (Cloudflare Registrar recommended, ~$10/yr for a `.com`). DNS managed via Cloudflare or the registrar; an `A`/`AAAA` record points to the Fly app's anycast IP.
- **Frontend:** built via `vite build`, served as static assets by the Express backend from `apps/web/dist`. Single deploy, single domain — no CORS hop for production traffic, no separate CDN needed in Phase 1.

### 14.1 Secrets

| Name | Purpose |
|---|---|
| `ANTHROPIC_API_KEY` | Claude |
| `OPENAI_API_KEY` | Whisper + TTS |
| `DATABASE_URL` | Postgres |
| `SESSION_TOKEN_SECRET` | HMAC for token integrity check |
| `AUDIO_TTL_DAYS` | Default 7 |
| `AUDIO_MAX_BYTES` | Default `5 * 1024 * 1024 * 1024` |
| `SESSION_TTL_DAYS` | Default 7 |
| `RATE_LIMIT_LLM_PER_HOUR` | Default 30 |
| `RATE_LIMIT_AUDIO_PER_MIN` | Default 5 |
| `MOCK_LLM` | `1` for offline dev/tests |

All managed via the platform's secret store. A `.env.example` lists every required var. Never committed.

### 14.2 Local Dev

- `docker compose up` starts Postgres.
- `npm run dev` (root) runs both apps via `concurrently`.
- Real API keys via gitignored `.env` files, or `MOCK_LLM=1` for fully offline work.

### 14.3 CI

- **GitHub Actions.** On PR: lint (eslint), typecheck, unit + integration tests.
- **On merge to `main`:** auto-deploy to Fly/Render.
- **Nightly:** E2E suite against the preview deployment.

## 16. Security Posture (Phase 1)

### 15.1 General

- Session tokens: 22-char base62, CSPRNG, ≥130 bits entropy. Bearer credentials.
- HTTPS-only in production (HTTP → HTTPS redirect at the Fly proxy).
- Audio file serving (`GET /audio/:id.mp3`) verifies the requesting session owns the parent message — direct guessing is infeasible.
- Resume binaries processed in-memory and discarded; only extracted text + parsed JSON hit the DB.
- CORS allowlist: production origin + `localhost:5173`. No wildcards.
- Input size caps: resume ≤5 MB, JD ≤50 KB, audio chunk ≤25 MB. Enforced before LLM calls.

### 15.2 Secrets & API Key Management

**Key model:** Operator (Tzahi) provides Anthropic and OpenAI API keys. All user traffic runs on these keys. Users have zero key setup. BYOK is explicitly deferred to Phase 2.

**Where keys live:**

| Location | What's there | Who can read |
|---|---|---|
| Local laptop `apps/api/.env` | Real keys for dev | Local user; filesystem perms only. Not in git. |
| Git repo | `.env.example` only — placeholder strings | Anyone with repo access. No real secrets. |
| Fly secrets store | Real keys, encrypted at rest by Fly | Operator via `fly` CLI auth; running container as env vars |
| Express process memory | Real keys via `process.env` | The process and anyone with container shell access |
| Browser / frontend | **Never.** | — |
| Postgres | **Never** in Phase 1. (Phase 2 BYOK: per-user keys encrypted with a server-held master key.) | — |

**How keys are loaded:**

- Dev: `dotenv` loads `apps/api/.env` into `process.env` at startup.
- Prod: keys set once via `fly secrets set ANTHROPIC_API_KEY=…`. Fly injects them as env vars; no `.env` file exists on the production machine.
- CI: tests run with `MOCK_LLM=1`; no real keys needed.

**Required `.gitignore` entries (committed from day one):**

```
.env
.env.*
!.env.example
/data/
node_modules/
dist/
```

**Hardening:**

- **Pre-commit hook** (`lefthook` or `husky` + `lint-staged`) runs `gitleaks protect --staged` to block commits containing strings that look like API keys. Catches accidental paste-into-code.
- **Pino redaction:** the logger redacts `*.apiKey`, `*.token`, `authorization` headers, `process.env.*_KEY`, and resume/message bodies. No key value ever lands in logs.
- **No env echo in any script.** CI scripts never `echo $SECRET`; tests never `console.log(process.env)`.
- **Response sanitation:** the error handler never includes `process.env`, `req.headers`, or unredacted request bodies in error responses.

**Threat model — realistic ways a key leaks, and the mitigation in place:**

| Threat | Mitigation |
|---|---|
| Accidentally committing `.env` | `.gitignore` + `gitleaks` pre-commit hook |
| Logging the key | Pino redaction list + lint rule banning `console.log(process.env)` |
| Returning the key in an API response | Error handler whitelist; integration test verifies no key-shaped string ever appears in any response body |
| Running container compromised | 2FA on Fly account; rotate keys quarterly |
| Vendor account compromised | 2FA on Anthropic, OpenAI; hard spending caps in vendor dashboards bound the blast radius |
| Key leaked in any way | Rotate immediately at the vendor dashboard; redeploy via `fly secrets set`; no code change needed |

**Vendor-level spending caps (set on day one, regardless of estimated usage):**

- Anthropic: $200/month hard cap, email alert at $100.
- OpenAI: $100/month hard cap, email alert at $50.
- These bound worst-case damage from a leaked key to the cap, not "unlimited."

**Rotation policy:**

- Routine rotation every 90 days (calendar reminder).
- Immediate rotation on any of: laptop compromise, repo accidentally made public, suspicious activity on vendor dashboard, departure of anyone with prod access.

### 15.3 Phase 2 BYOK Considerations (Not in Phase 1, documented for evolution)

When BYOK arrives: per-user keys stored in a new `user_credentials` table, encrypted with AES-256-GCM using a server-held master key (held in Fly secrets, never in DB). Master key rotation strategy: dual-key for the rotation window, re-encrypt rows on access. Users can paste/replace/delete their key from the UI; deletion is immediate and audited.

## 17. Cost & Infrastructure Estimates

All figures are **list-price estimates as of the spec date** (2026-05-28). They assume the model selections, prompt caching, and rate limits defined elsewhere in this spec. They will drift over time and should be revalidated when costs become operationally relevant.

### 16.1 Vendor unit rates (snapshot)

| Service | Rate |
|---|---|
| Claude Sonnet 4.6 input / output / cached read | $3.00 / $15.00 / $0.30 per 1M tokens |
| Claude Opus 4.7 input / output | $15.00 / $75.00 per 1M tokens |
| OpenAI Whisper STT | $0.006 per minute of audio |
| OpenAI `tts-1` | $15.00 per 1M characters |

### 16.2 Per-operation costs

- Resume parse (Sonnet): **~$0.01** per resume.
- Gap analysis (Opus): **~$0.27** per role analyzed.
- Text interview turn (Sonnet, prompt caching active): first turn ~$0.02, subsequent turns **~$0.008** each.
- Voice interview turn (Sonnet + Whisper + TTS): **~$0.010** per turn.
- End-of-interview feedback (Opus): **~$0.45** per completed interview. *(Tuning knob: switching to Sonnet drops this to ~$0.09 with some loss of feedback depth.)*
- Q&A tutor turn (Sonnet): **~$0.022**.

### 16.3 Monthly cost — 6 active users (canonical Phase 1 estimate)

**Usage assumptions:** 6 users, each doing ~3 mock interviews/week, ~60 turns/interview, half voice/half text; 5 gap analyses/user/month; 30 Q&A questions/user/month.

| Line item | Detail | $/month |
|---|---|---:|
| Fly compute | `shared-cpu-1x`, 1 vCPU / 1 GB | $5 |
| Fly volume | 10 GB at `/data` | $1.50 |
| Fly Postgres | Development cluster | $15 |
| Domain | `.com` amortized | $1 |
| **Infra subtotal** | | **$22** |
| Claude Sonnet — interview turns | ~7,200 turns × $0.008 | $58 |
| Claude Opus — gap + feedback | 30 × $0.27 + 72 × $0.45 | $41 |
| Claude Sonnet — Q&A | 180 × $0.022 | $4 |
| Whisper STT | ~600 min × $0.006 | $4 |
| OpenAI TTS | ~3,600 voice turns × ~500 chars avg | $27 |
| Resume parsing | ~30 resumes × $0.01 | $0.30 |
| **AI subtotal** | | **$134** |
| **MONTHLY TOTAL** | | **~$156** |

### 16.4 Sensitivity ranges

- **Light usage (6 users, mostly casual):** $80–120/mo
- **Steady usage (assumptions above):** $140–165/mo
- **Heavy usage (everyone practicing daily before real interviews):** $250–350/mo

### 16.5 Cost-driver observations

- **AI APIs are ~86% of every dollar.** Hosting choice barely moves the needle at Phase 1 scale.
- **Largest controllable cost is end-of-interview feedback** (~$41/mo at 6 users). The Sonnet-vs-Opus tradeoff for this prompt is the highest-leverage tuning knob.
- **TTS scales with output length, not turn count.** The `interviewer` prompt is tuned to keep interviewer turns short and conversational; lecturing would balloon TTS cost.

### 16.6 Spending guardrails (mandatory on day one)

These are in addition to in-app rate limits (Section 9.2):

- Anthropic monthly cap: **$200**, alert at $100.
- OpenAI monthly cap: **$100**, alert at $50.
- App-level rate limit per session token (already in spec): 30 LLM calls/hr + 5 audio uploads/min, bounds worst-case session burn at ~$5/hr.

Combined, the worst-case monthly spend even with a leaked session token is capped at $300 ($200 + $100 vendor limits), not unbounded.

## 18. Repo Conventions

- **TypeScript strict mode** everywhere. `any` requires a `// reason: ...` comment.
- **Shared types in `packages/shared`.** Both apps import from there. Runtime validation at API boundary via Zod (request bodies server-side, response shapes client-side).
- **Prompts in their own files** (Section 10), with version strings.
- **Thin route handlers, fat services.** `routes/` → `services/` → `repositories/`. Each file focuses on one concern.
- **Lint + format via eslint + prettier.** Single shared config at repo root.
- **Git from day one.** `git init` is part of the first scaffold step. `.gitignore` covers `node_modules`, `.env*` (except `.env.example`), `dist/`, `/data/`, and editor folders.

## 19. Phase 2+ Evolution Path

Designed so the following are additive, not rewrites:

- **Real auth (email magic link or OAuth):** add `users` table, make `sessions.user_id` nullable, claim anonymous sessions on signup. No schema break.
- **Multi-region / scale-out:** swap local disk for S3-compatible storage. `audio_url` already abstracts location.
- **WebSocket voice:** today's SSE + multipart is fine for v1. WS upgrades the voice-turn endpoint without changing the prompt/conversation contract.
- **Languages beyond Hebrew/English:** language directive in `interviewer.ts` is already parameterized.
- **Custom voices (ElevenLabs):** one config change in `openai.ts`.
- **Mobile client:** REST/SSE API is the contract; React Native client reuses `packages/shared` types.
- **Improvement loop:** `usage_log` + `messages` already structured for offline analysis. Add a thumbs-up/down per assistant turn.

## 20. Out of Scope (Phase 1, Explicit)

- Real accounts, OAuth, magic links.
- Payments / billing.
- Real-time barge-in for voice.
- Vector search / RAG.
- Multi-region or HA deploy.
- Mobile apps.
- Languages other than Hebrew/English.
- Long-term retention beyond 7 days.
- Live-interview assistance (real interviews with employers). The product is explicitly a coaching tool; this is a product-policy boundary, not a technical one.

## 21. Open Items Tracked for Phase 1 Implementation

- Exact Claude model selection (Sonnet 4.6 vs Opus 4.7 per prompt) — to be tuned during build, based on latency vs quality trade-offs per mode. Voice mode is most latency-sensitive.
- TTS voice selection (`nova` vs `alloy` vs others) for Hebrew quality — A/B during build.
- Concrete pricing values for the cost meter — initialize from Anthropic and OpenAI published rates at build time; update via config file when rates change.
