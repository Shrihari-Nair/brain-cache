# 13 — Glossary and Reference

## 13.1 Glossary (alphabetical; project-specific meaning where it differs)
*Evidence labels: see [00 §0.1](00-learning-roadmap.md#01-how-to-read-the-evidence-labels). Entries without a label are definitions ([Standard knowledge]) plus a pointer to where the repo uses them.*

| Term | Meaning | Where it appears |
|---|---|---|
| **Agent (LiveKit)** | A Python worker that joins a room as a participant and runs the voice pipeline | `mockmind/agent/` |
| **Agent dispatch** | Instruction (embedded in the token as `RoomAgentDispatch`) telling LiveKit to send a named agent (`mockmind-interviewer`) into the room | `routers/session.py` |
| **ASGI** | Python async web-server interface used by FastAPI/Uvicorn | [02a](02a-backend-and-data-technologies.md) |
| **Autosave** | Debounced write of `session_state:{id}` to Redis during the interview (1.0 s delay, `_AUTOSAVE_DELAY_S`) | `session_state.py` `_schedule_autosave` / `_autosave` (L93–110) |
| **Backoff** | Waiting longer between retries (1 s, 2 s here: 3 attempts, `base_delay=1.0`) | `llm_retry.py` `retry_async` |
| **Bind mount** | Host folder mapped into a container; Compose mounts `.:/app` for `backend` and `agent` | `docker-compose.yml` |
| **Candidate memory** | Opt-in, per-email record of weak categories, asked questions and notes, used only at *preparation* time | `memory_store.py`, `memory_updater.py` |
| **Category** | Question classification enum `QuestionCategory`: `behavioral`, `technical`, `situational`, `resume_deep_dive`, `culture_fit`, `closing` (never `"coding"`; coding is `question_type`, see [07](07-engineering-challenges.md)) | `models/interview_plan.py` L6–12, L42 |
| **CORS** | Browser rule restricting cross-origin reads; allowlist from `CORS_ORIGINS` | `main.py` |
| **Constrained agent** | LLM with tools inside a code-controlled flow | [02b](02b-ai-and-llm-deep-dive.md) |
| **Data channel** | LiveKit message path between participants; topics `coding_challenge`, `code_submission` | `agent.py`, frontend |
| **Deterministic check** | Rule-based eval check on a transcript (no LLM) | `evals/checks.py` |
| **Endpointing** | Deciding the candidate has finished speaking (delay + model) | [02c](02c-real-time-voice-and-livekit.md) |
| **Eval (harness)** | Persona-driven simulated interviews scored by checks and an LLM judge | `mockmind/evals/` |
| **Follow-up cap** | Max 2 follow-ups per question (`MAX_FOLLOW_UPS = 2`, `session_state.py` L13), enforced by a prompt note and a counter (the model can still ignore it; "soft") | `session_state.py` `follow_up_policy_note` (L137–155) |
| **Generation (Langfuse)** | Trace observation type for one LLM call | `services/tracing.py` `traced_generation` |
| **Hook (`on_user_turn_completed`)** | LiveKit callback after a user turn, used here to inject the policy note | `interviewer.py` |
| **Ideal answer points** | Answer key per question inside the plan | `InterviewQuestion` |
| **Interview plan** | Prepared JSON: questions, levels, coding specs, opening message | `interview_plan:{id}` |
| **JWT** | Signed token; here a LiveKit access token with grants | `get_token` |
| **Judge (LLM judge)** | LLM scoring a transcript against a rubric | `evals/judge.py` |
| **Lifespan** | FastAPI startup/shutdown hook; here pings Redis (fail fast) and closes the client | `main.py` L15–18 |
| **Langfuse / OTel** | Optional tracing backend / vendor-neutral tracing standard | `tracing.py` |
| **LiveKit** | WebRTC infrastructure (rooms, tracks, data channels) | [02c](02c-real-time-voice-and-livekit.md) |
| **Persona** | Scripted simulated candidate style (strong, vague, prompt_injector, coder, buggy_coder) | `evals/personas.py` |
| **Piston** | Open-source sandbox executing code in many languages | `docker-compose.yml`, `code_executor.py` |
| **Preemptive generation** | Starting LLM/TTS before the turn is confirmed finished, to cut latency | [02c](02c-real-time-voice-and-livekit.md) |
| **Pydantic** | Python data validation library | models |
| **Preflight (CORS)** | Browser `OPTIONS` request that asks permission before a cross-origin JSON `POST` | handled by `CORSMiddleware` |
| **Preflight (evals)** | Script that checks Piston is reachable before coding evals | `evals/preflight.py` |
| **Report status** | `pending`, `ready`, `failed` in `interview_report_status:{id}` | `report_store.py` |
| **RDB / AOF** | Redis snapshot / append-only-log persistence modes; image default here is RDB only (**[Measured]**, [02a](02a-backend-and-data-technologies.md) §2a.4) | `mockmind-redis-1` |
| **Retry endpoint** | Re-runs report generation after failure | `routers/session.py` `retry_report` |
| **Room** | LiveKit session space, named `interview_{session_id}` | `get_token` |
| **Session id** | UUID tying plan, state, report, room | everywhere |
| **SFU** | Selective forwarding unit: relays media between WebRTC peers without mixing | LiveKit ([02c](02c-real-time-voice-and-livekit.md)) |
| **Span / trace** | Timed operation / tree of operations in OpenTelemetry | `agent.py` `_setup_otel_tracing` |
| **SSE** | Server-Sent Events; one-way HTTP streaming (`data: {...}` + blank line) | `prepare.py` `_sse` |
| **STT / TTS** | Speech-to-text (Soniox) / text-to-speech (Cartesia) | `agent.py` |
| **Thinking budget** | Gemini setting that allows hidden reasoning tokens (`-1` dynamic, `0` off) | `question_generator.py`, `report_generator.py` |
| **Tool call** | Model-requested function invocation (`move_to_next_question`, …) | `interviewer.py` |
| **TTL** | Time-to-live (Redis expiry) | `setex` calls |
| **Worker / job (LiveKit)** | Long-running agent process / one interview assigned to it | `agent/agent.py` `entrypoint` |
| **VAD** | Voice-activity detection (Silero) | `agent.py` |
| **Workflow (LLM)** | Fixed pipeline of LLM calls | preparation |

## 13.2 Redis key reference

| Key | Writer | Reader | TTL |
|---|---|---|---|
| `interview_plan:{id}` | backend `prepare` | backend `get_token`, agent `load` | 4 h |
| `session_state:{id}` | backend `prepare` (empty state), then agent `save`/autosave (TTL refreshed each save) | backend `retry_report` (rebuilds the report from the stored transcript); agent does not reload it on reconnect **[Verified]** | 4 h |
| `interview_report:{id}` | `run_report_pipeline` | `get_report`, `retry_report` (existence check) | 24 h |
| `interview_report_status:{id}` | `set_status` (`pending`/`ready`/`failed`) | `get_report`, `retry_report` | 24 h |
| `candidate_memory:{sha256(email)}` | `memory_store.save` | `prepare` (opt-in) | 1 y |

TTLs in seconds: plan/state 14 400 (`prepare.py` L186–198, `session_state.py` `save`); report and status 86 400 (`report_store.py`); memory 31 536 000 (`memory_store.py`). Inspect with `docker compose exec redis redis-cli ttl <key>` / `get <key>` (the Compose service is named `redis`; port 6379 is also published to the host).

## 13.3 HTTP API reference (from `backend/routers/`)

| Method / path | Purpose | Notes |
|---|---|---|
| `POST /api/prepare` | multipart form (`resume` file, `job_description`, optional `use_memory`, `candidate_email`, `interview_type`, `difficulty`, `length`, `focus_weak_areas`) → SSE progress, final `complete` with `session_id` | limits 5 MB / 30 000 chars resume text (truncated) / 15 000 chars JD (rejected); always HTTP 200, errors as `stage:"error"` events |
| `POST /api/token` | body `{session_id, participant_name}` → `{token, ws_url, room_name, identity, session_id}`; token TTL 2 h, dispatches `mockmind-interviewer` | 404 if plan expired; 422 if body invalid; 500 if `LIVEKIT_*` unset |
| `GET /api/report/{session_id}` | report JSON | 200 report; 502 `{status:"failed"}`; 404 not ready/unknown; `get_report` (decorator L87) |
| `POST /api/report/{session_id}/retry` | re-run pipeline; returns 202 `{status:"pending"}` or `{status:"ready"}` | 404 no session state; 409 empty transcript; no-op while pending; `retry_report` (decorator L112) |
| `DELETE /api/candidate-memory` | body `{email}` (`ForgetMeRequest`) → delete the hashed-email key; 204 | `routers/memory.py` L13–18 |
| `GET /health` | static `{"status":"ok"}` | no dependency probes |

Paths verified from the router decorators; **[Measured]** earlier: `/openapi.json` lists these 6 operations. FastAPI also serves `GET /docs` (Swagger UI) and `GET /openapi.json` automatically. Status-code meanings: 200 OK, 202 accepted (async work started), 204 no content, 404 not found/expired, 409 conflict (nothing to evaluate), 422 validation failure (framework), 502 upstream/generation failure.

## 13.3.1 SSE stage reference (`/api/prepare`)
| stage | progress | meaning |
|---|---|---|
| `parsing` | 5 → 15 | read/validate/extract the resume |
| `resume_analysis` | 20 → 45 | Gemini → `CandidateProfile` |
| `jd_analysis` | 50 → 65 | Gemini → `JobProfile` |
| `question_gen` | 70 (→72) → 90 | Gemini → `InterviewPlan` questions |
| `complete` | 100 | plan stored; payload `data` has `total_questions`, `duration_minutes`, `job_title`, `candidate_name` |
| `error` | 0 | user-safe message; HTTP status was already 200 |

## 13.3.2 Ports and URLs (host side)
`8000` backend (`/docs`, `/health`) · `3000` Next.js dev server (README) · `3002` Compose frontend (→ container 3000) · `3001` Langfuse web · `9090` MinIO API (→ 9000) · `2000` Piston · `6379` Redis. See the CORS caveat in [00 §0.4](00-learning-roadmap.md).

## 13.3.3 Key constants and models
| Item | Value | Source |
|---|---|---|
| `MAX_RESUME_BYTES` / `MAX_RESUME_CHARS` / `MAX_JD_CHARS` | 5 MiB / 30 000 / 15 000 | `prepare.py` L56–59 |
| `LENGTH_PRESETS` | `quick`=8, `standard`=`DEFAULT_TOTAL_QUESTIONS`=16 | `prepare.py` L42, `question_generator.py` L42 |
| `MAX_FOLLOW_UPS` | 2 | `session_state.py` L13 |
| LiveKit token TTL / room / identity | 2 h / `interview_{id}` / `candidate_{id[:8]}` | `session.py` |
| Piston `run_timeout` / HTTP timeout | 3000 ms / 30 s | `code_executor.py` |
| Retry policy | 3 attempts, 1 s then 2 s | `llm_retry.py` |
| Gemini model used in code | `gemini-2.5-flash` (backend, agent, evals; **[Verified]** by grep) | various |
| Frontend polling / agent-join timeout | 4 s / 25 s | `report/[sessionId]/page.tsx` L81; `interview/[sessionId]/page.tsx` L368 |

## 13.4 Verified commands

| Command | Status |
|---|---|
| `cd mockmind && backend/.venv/bin/python -m pytest backend/tests -q` | **[Measured]** 53 passed, 1 skipped (re-run during audit, ~1.4 s) |
| `cd mockmind && backend/.venv/bin/python agent/agent.py --help` | **[Measured]** prints CLI and the `livekit.plugins.turn_detector` deprecation warning |
| `python -m pytest backend/tests -q` (CI form, run from `mockmind/`) | in `ci.yml`; the CI form was not run as-is; the venv-python form above gives the same suite |
| `cd mockmind/frontend && npx tsc --noEmit`, `npm run build` | in `ci.yml`; **[Not run]** here |
| `python evals/run_eval.py --persona all --runs 3 --concurrency 3 --min-pass-rate 0.8` | in `evals.yml`; **[Not run]** (needs `GOOGLE_API_KEY`, Piston for coding personas) |
| `docker compose config -q` | **[Measured]** exit 0 (warns that the top-level `version` key is obsolete) |
| `docker compose up` | in `docker-compose.yml`; **[Not run]** (some services were already running on the analysis machine) |

## 13.5 Environment variables (names only; values never recorded)
From `mockmind/.env.example` (**[Verified]**, names only): `LIVEKIT_URL`, `LIVEKIT_API_KEY`, `LIVEKIT_API_SECRET`, `GOOGLE_API_KEY`, `CARTESIA_API_KEY`, `SONIOX_API_KEY`, `REDIS_URL`, `PISTON_URL`, `CORS_ORIGINS`, and a set of `LANGFUSE_*` variables (host, public/secret keys, and the self-hosted Langfuse stack's salt/encryption/auth/database/object-store settings). Values are never recorded here. Required vs defaulted, and which process reads each, is tabulated in [02e §2e.2](02e-infra-and-tooling.md). Frontend variables (`frontend/.env.local.example`, Compose build args): `NEXT_PUBLIC_LIVEKIT_URL`, `NEXT_PUBLIC_BACKEND_URL` (baked into the browser bundle at build time, so public by design). Compose substitutes `${LIVEKIT_URL}` and the `${LANGFUSE_*}` names from `.env`.

## 13.6 Reading order for a new engineer
[00](00-learning-roadmap.md) → [01](01-fundamentals.md) → [03](03-system-architecture.md) → [05](05-end-to-end-workflows.md) → [04](04-codebase-walkthrough.md) → [02a–e](02-technology-deep-dives.md) → [06](06-core-concepts-and-implementation.md) → [07](07-engineering-challenges.md) → [09](09-testing-evaluation-and-observability.md) → [11](11-interview-preparation.md).
