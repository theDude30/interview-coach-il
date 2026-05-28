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
| DB | Postgres (managed: Neon or Fly Postgres) |
| Audio storage (Phase 1) | Local disk under `/data/audio` |
| Data TTL | 7 days for everything (audio + text); session activity bumps it |
| Audio cap | 5 GB global, oldest-first eviction |
| Rate limit | 30 LLM calls/hr, 5 audio uploads/min, per session token |
| Cost meter | Visible in UI, server-computed |
| Mic UX | Toggle (click to start, click to stop). Spacebar shortcut. |
| Deploy target | Fly.io or Render |
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

## 4. Session Model

The app is "anonymous but persistent." On first visit, the backend issues a session token (22-char base62, ≥130 bits of entropy from CSPRNG). The token is stored in browser localStorage and sent as `X-Session-Token` on every API call. The URL form `?s=<token>` re-attaches a session — this is how a user shares progress across devices or with a friend.

**Implication:** the token is a bearer credential. Anyone with it has full access to that session's data. Acceptable for Phase 1, replaced by real auth in Phase 2.

**Session lifecycle:**
- `POST /sessions` issues a fresh token.
- Every authenticated request bumps `sessions.last_active_at`.
- The hourly GC deletes sessions inactive for more than `SESSION_TTL_DAYS` (default 7), cascading to all dependent rows and audio files.
- A user returning after expiry lands on an empty session and sees a banner: "This session expired; starting fresh."

## 5. Data Model (Postgres)

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

## 6. Retention & Garbage Collection

A single in-process hourly job (`node-cron`) handles all TTL work:

1. **Audio TTL.** Delete every file under `/data/audio` older than `AUDIO_TTL_DAYS` (default 7). For each deleted file, clear `messages.audio_url`.
2. **Audio cap.** If `/data/audio` total size exceeds `AUDIO_MAX_BYTES` (default 5 GB), delete oldest files first until under the cap.
3. **Session TTL.** Delete every `sessions` row where `last_active_at < now() - SESSION_TTL_DAYS`. Cascade deletes handle the rest of the data.

UX implications:
- A returning user whose audio expired but session is still active sees transcripts with a small "audio expired" note next to each turn. Replay works for text.
- A returning user whose session expired lands on a fresh session, sees a one-line banner.

## 7. The Four User Modes

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

## 8. Backend API Surface

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

## 9. Prompt Architecture

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

## 10. LLM/STT/TTS Client Wrappers

`apps/api/src/clients/claude.ts`:
- Wraps `@anthropic-ai/sdk`.
- Exponential backoff retry on 429 / 5xx (max 3 attempts).
- Logs every call to `usage_log` with token counts and computed USD cost (rates in `apps/api/src/config/pricing.ts`).
- Streaming abstraction yields normalized events.
- `MOCK_LLM=1` swaps to an in-memory fake returning canned responses keyed by prompt fingerprint.

`apps/api/src/clients/openai.ts`: same shape for Whisper and TTS.

## 11. Error Handling — User-Visible Contract

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

## 12. Testing Strategy

**Unit (Vitest):** prompt template renderers (snapshot tests), pure helpers (conversation summarizer, cost calculator, rate limiter, resume parser post-processing), React component logic (RTL handling, hint slider, cost meter).

**Integration (Vitest + supertest):** every REST route end-to-end against a real Postgres via testcontainers. LLM/STT/TTS calls stubbed at the client boundary using the same fakes the `MOCK_LLM` flag enables. Critical paths: session lifecycle, resume upload → parse → fetch, gap analysis SSE stream completes, full text interview turn, full voice interview turn (audio fake returns fixed mp3), TTL GC actually deletes expired rows + files, rate limiter returns 429 after threshold.

**E2E (Playwright, gated):** three scenarios, runnable with `E2E=1` against real APIs. Not in default CI.
1. New session → upload resume → run gap analysis → see report.
2. New session → start text interview → complete one round → end → see feedback.
3. New session → start voice interview → speak one turn → hear reply.

**LLM evals (`apps/api/evals/`):** scripted harness runs the interviewer prompt against ~20 transcripts using Claude as judge. Scores behavioral consistency (one-question-at-a-time, language adherence, hint compliance). Results checked into the repo as JSON. On-demand, not on every commit.

## 13. Observability

- **Logs (pino):** `request_id`, `session_id`, `route`, `prompt_version`, `latency_ms`, token counts, cache hit counts. Resume text and message bodies redacted.
- **Metrics (`prom-client` at `/metrics`):** request counts, p50/p95/p99 per route, LLM error rates, GC throughput, `/data/audio` size.
- **`usage_log` table** drives the in-UI cost meter and ad hoc "which session is burning budget" queries.
- **`audit_log` table** for refusals, rate-limit hits, parse failures.
- **No external APM in Phase 1.** Platform's built-in log viewer is enough.

## 14. Deployment

- **Target:** Fly.io or Render. Both support persistent volumes (needed for `/data/audio`), managed Postgres, and TLS. Vercel is excluded — persistent disk and long SSE streams don't fit.
- **Single backend instance:** 1 vCPU, 1 GB RAM.
- **Postgres:** managed (Neon free tier or Fly Postgres).
- **Volume:** 10 GB mounted at `/data`.
- **Frontend:** built via `vite build`, served as static assets by the Express backend from `apps/web/dist`.

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

## 15. Security Posture (Phase 1)

- Session tokens: 22-char base62, CSPRNG, ≥130 bits entropy. Bearer credentials.
- HTTPS-only in production.
- Audio file serving (`GET /audio/:id.mp3`) verifies the requesting session owns the parent message.
- Resume binaries processed in-memory and discarded; only extracted text + parsed JSON hit the DB.
- CORS allowlist: production origin + `localhost:5173`. No wildcards.
- Input size caps: resume ≤5 MB, JD ≤50 KB, audio chunk ≤25 MB. Enforced before LLM calls.
- API keys never leave the server.

## 16. Repo Conventions

- **TypeScript strict mode** everywhere. `any` requires a `// reason: ...` comment.
- **Shared types in `packages/shared`.** Both apps import from there. Runtime validation at API boundary via Zod (request bodies server-side, response shapes client-side).
- **Prompts in their own files** (Section 9), with version strings.
- **Thin route handlers, fat services.** `routes/` → `services/` → `repositories/`. Each file focuses on one concern.
- **Lint + format via eslint + prettier.** Single shared config at repo root.
- **Git from day one.** `git init` is part of the first scaffold step. `.gitignore` covers `node_modules`, `.env*` (except `.env.example`), `dist/`, `/data/`, and editor folders.

## 17. Phase 2+ Evolution Path

Designed so the following are additive, not rewrites:

- **Real auth (email magic link or OAuth):** add `users` table, make `sessions.user_id` nullable, claim anonymous sessions on signup. No schema break.
- **Multi-region / scale-out:** swap local disk for S3-compatible storage. `audio_url` already abstracts location.
- **WebSocket voice:** today's SSE + multipart is fine for v1. WS upgrades the voice-turn endpoint without changing the prompt/conversation contract.
- **Languages beyond Hebrew/English:** language directive in `interviewer.ts` is already parameterized.
- **Custom voices (ElevenLabs):** one config change in `openai.ts`.
- **Mobile client:** REST/SSE API is the contract; React Native client reuses `packages/shared` types.
- **Improvement loop:** `usage_log` + `messages` already structured for offline analysis. Add a thumbs-up/down per assistant turn.

## 18. Out of Scope (Phase 1, Explicit)

- Real accounts, OAuth, magic links.
- Payments / billing.
- Real-time barge-in for voice.
- Vector search / RAG.
- Multi-region or HA deploy.
- Mobile apps.
- Languages other than Hebrew/English.
- Long-term retention beyond 7 days.
- Live-interview assistance (real interviews with employers). The product is explicitly a coaching tool; this is a product-policy boundary, not a technical one.

## 19. Open Items Tracked for Phase 1 Implementation

- Exact Claude model selection (Sonnet 4.6 vs Opus 4.7 per prompt) — to be tuned during build, based on latency vs quality trade-offs per mode. Voice mode is most latency-sensitive.
- TTS voice selection (`nova` vs `alloy` vs others) for Hebrew quality — A/B during build.
- Concrete pricing values for the cost meter — initialize from Anthropic and OpenAI published rates at build time; update via config file when rates change.
