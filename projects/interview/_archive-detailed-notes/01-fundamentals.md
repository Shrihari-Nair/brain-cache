# 01 — Fundamentals: the prerequisites for this codebase

> **Read this if** any term in [00](00-learning-roadmap.md) was unfamiliar, or you want a refresher before the deep dives. Each section is short, builds from first principles, and ends with **where the concept appears in MockMind**. Concepts the repository does *not* use are marked **(not used here)** so you don't waste time on them.
> **Labels:** see [00 §0.1](00-learning-roadmap.md#01-how-to-read-the-evidence-labels). Everything in this chapter that isn't tied to a source reference is **[Standard knowledge]**.

---

## 1.1 Programs, modules, functions, classes, imports

- A **program** is instructions a computer runs. A **process** is a running instance of a program with its own memory.
- A **function** packages reusable steps: `def add(a, b): return a + b`.
- A **class** bundles data (fields) and behaviour (methods) into a blueprint; an **object** is an instance.
- A **module** is a file of code; **`import`** makes another module's names available. A **package** is a folder of modules.
- A **constant** (ALL_CAPS by convention in Python) is a value not meant to change.

**Where used:** the Python backend is a package `backend` (`mockmind/backend/`), imported by the agent too (`from backend.models.interview_plan import InterviewPlan` in `mockmind/agent/session_state.py`). That cross-import is why the agent's `requirements.txt` repeats the backend's libraries. **[Verified]**

### Decorators and generators (needed to read FastAPI code)
- A **decorator** (`@something` above a function) wraps it to add behaviour. FastAPI uses `@router.post("/api/token")` to register a function as an HTTP endpoint (`mockmind/backend/routers/session.py` L30 — the decorator line; `get_token` itself is L31–84).
- A **generator** uses `yield` to produce values one at a time, pausing in between. An **async generator** (`async def` + `yield`) can also `await`. FastAPI turns an async generator into a streamed response (`event_stream()` in `mockmind/backend/routers/prepare.py`, L117–223).

## 1.2 Types, schemas and validation

- A **type** says what kind of value something is (`int`, `str`, `list[str]`). Python's **type hints** (`x: int`) are documentation plus tooling hints; Python does **not** enforce them at runtime by default.
- A **schema** describes the *shape* of data (fields and their types). **Validation** checks real data against a schema.
- **Pydantic** (used here) turns type hints into runtime validation: `CandidateProfile(**data)` raises an error if `years_of_experience` is `"lots"`.
- **TypeScript** adds static types to JavaScript; it is checked at build time (`tsc`) and erased before the browser runs the code.

**Worked example from this repo [Verified by running the real models]:** an LLM is asked for questions, and replies with `{"category": "coding", ...}`. `InterviewQuestion.category` is an enum (`QuestionCategory`, `models/interview_plan.py` L6–12) that does not include `"coding"`, so `InterviewQuestion(**data)` raises `ValidationError` at once, *at the boundary*, instead of letting a bad value travel into Redis and crash the voice agent later. That precise failure happened in practice (commit `b2ba44e`, message: "crashing generate_questions() with an uncaught ValueError on roughly every other real call"; see [07](07-engineering-challenges.md) §7.1). Note two Pydantic defaults that surprise newcomers: unknown fields are ignored, and `"4"` is coerced to `4` (details in [02a](02a-backend-and-data-technologies.md) §2a.3).

**Where used:** `mockmind/backend/models/interview_plan.py` (all models), `mockmind/backend/models/candidate_memory.py`, `mockmind/frontend/types/interview.ts`. **Why it matters:** AI output is untrusted text; parsing it into Pydantic models is the boundary that rejects malformed data. **[Verified]**

## 1.3 JSON, HTTP and APIs

- **JSON** is a text format for structured data (objects `{}`, arrays `[]`, strings, numbers, booleans, `null`).
- **HTTP** is the request/response protocol of the web. A request has a **method** (`GET`, `POST`, `DELETE`…), a **URL** (host, port, path), **headers**, and an optional **body**. A response has a **status code**, headers, and a body.
- An **API endpoint** is a method + path that a program can call. **REST** style names endpoints after resources (`GET /api/report/{id}`).
- **Status codes used here:** 200 OK, 202 Accepted (work started), 204 No Content, 400 Bad Request, 404 Not Found, 409 Conflict, 422 Unprocessable Entity (FastAPI validation), 502 Bad Gateway (upstream failure), 500.
- **Multipart form data** sends files plus fields in one request (used for the resume upload).
- **CORS** (Cross-Origin Resource Sharing): a *browser* rule — a page may only read responses from a different origin if that server allows it via headers. `curl` ignores it.

**One request, line by line.** The browser calls `POST http://localhost:8000/api/token` with header `Content-Type: application/json` and body `{"session_id": "<uuid>"}` (`TokenRequest`, `routers/session.py` L25–27; `participant_name` is the other field). The server replies `200` with JSON `{token, ws_url, room_name, identity, session_id}`. If the body is missing `session_id`, FastAPI answers `422` before our code runs; if the plan key has expired, our code answers `404 "Session not found..."`. A third outcome [Verified from code, not run]: if `LIVEKIT_API_KEY/SECRET/URL` are unset, `os.environ[...]` raises `KeyError` and the server returns `500`. Knowing *who* produces each status (framework, our code, crash) is the key debugging skill.

**Preflight.** Because the frontend (port 3000) and backend (port 8000) are different origins, browsers first send an `OPTIONS` *preflight* for JSON `POST`s; `CORSMiddleware` answers it. The allowed origins come from the `CORS_ORIGINS` environment variable (comma-separated, default `http://localhost:3000`), with `allow_credentials=True`, `allow_methods=["*"]`, `allow_headers=["*"]` (`backend/main.py` L23–29).

**Where used:** endpoints in `mockmind/backend/routers/*.py`; the allowed-origins list in `mockmind/backend/main.py` (`CORSMiddleware`, `CORS_ORIGINS`). **[Verified]** Full contract tables: [02a](02a-backend-and-data-technologies.md) and [13](13-glossary-and-reference.md).

## 1.4 Streaming and real-time communication styles

| Style | How it works | In this repo |
|---|---|---|
| **Polling** | client asks repeatedly | report page → `GET /api/report/{id}` every 4 s (`mockmind/frontend/app/report/[sessionId]/page.tsx`) |
| **Server-Sent Events (SSE)** | server pushes lines `data: {...}\n\n` over one HTTP response | `/api/prepare` progress (`prepare.py` → `_sse()`) |
| **WebSocket** | persistent two-way connection | used *inside* LiveKit (not directly by our code) |
| **WebRTC** | browser real-time audio/video/data, mostly over UDP, encrypted | the live interview (via LiveKit) |

**Why SSE here, and why `fetch` instead of `EventSource`?** SSE is one-way and simple; but the browser's `EventSource` can only issue `GET` without a body, and we must `POST` a file, so the frontend reads the response body stream manually (`mockmind/frontend/hooks/usePreparation.ts`). **[Verified]**

## 1.5 Concurrency, async and processes

- **Concurrency**: many tasks in progress; **parallelism**: many tasks literally executing simultaneously on multiple CPU cores.
- A **thread** shares memory with its process; a **process** has separate memory.
- **`async`/`await`** (Python `asyncio`) gives *cooperative* concurrency on one thread: when a task `await`s a slow operation (network), the **event loop** runs another task. Ideal for **I/O-bound** work; **CPU-bound** or **blocking** calls freeze the loop.
- A **background task** (`asyncio.create_task(...)`) runs without the caller waiting. Python keeps only a weak reference to tasks; hold a reference or the task can be garbage-collected mid-run.
- Python's **GIL** lets only one thread execute Python bytecode at a time, which is why CPU-heavy work often uses processes.

**Why a blocking call is dangerous, with this repo's example.** Imagine two users hit `/api/prepare` at the same moment. User A's request reaches `extract_resume_text()` (`routers/prepare.py` L129), a *plain synchronous function* that runs `pypdf` over the PDF. While it runs, the single event-loop thread is busy, so user B's request (and every `GET /health`) waits — there is no `await` for the loop to switch at. By contrast, `await asyncio.sleep(...)` inside `retry_async()` (`services/llm_retry.py` L10–33) *yields*, so other requests proceed during the backoff. How long the PDF parse actually blocks is **[Not measured]**; that it *can* block is **[Standard knowledge]** because the code never offloads it (no `asyncio.to_thread`/executor anywhere — verified by grep, see [02a](02a-backend-and-data-technologies.md) §2a.1).

**Fire-and-forget gotcha.** `asyncio.create_task(coro)` starts work but returns immediately; if you drop the returned object the task may be garbage-collected and silently stop. `routers/session.py` keeps a module-level set `_retry_tasks` (L109) and removes tasks with `add_done_callback(_retry_tasks.discard)` (L137–138) for exactly that reason. The agent-side report task (`interviewer.py` L149) does not store the handle; the related report-teardown risk is item D1 in [07](07-engineering-challenges.md).

**Where used:** every backend endpoint is `async def`; `asyncio.create_task` in `mockmind/backend/routers/session.py` (retry) and `mockmind/agent/interviewer.py` (report); one **OS process per live interview** in the LiveKit worker (**[Library source]** `livekit/agents/worker.py`, default `JobExecutorType.PROCESS` on Linux). Deep dive: [08](08-performance-reliability-and-scaling.md) §8.5.

## 1.6 Data storage, caching and Redis

- A **database** persists structured data (e.g. PostgreSQL, tables, SQL). **This application has no SQL database of its own** **[Verified]**: no ORM, no migrations, no SQL in the repo. (Langfuse's *own* containers use Postgres/ClickHouse internally.)
- A **cache / key–value store** maps a *key* to a *value* in fast memory. **Redis** is in-memory (optionally persisted). Commands used: `GET`, `SETEX key ttl value` (set with auto-expiry), `DEL`, `PING`.
- **TTL** (time-to-live) auto-deletes a key. **Idempotent** operations can repeat safely.
- **Hashing** (e.g. SHA-256) maps data to a fixed fingerprint; used to avoid raw emails in Redis keys.

**A tiny Redis session, as this app uses it.** `SETEX interview_plan:abc 14400 '{...json...}'` stores the JSON text for 4 h (14400 s = 4 × 3600). `GET interview_plan:abc` returns the string or *nil* (Python `None`). After 14400 s Redis deletes the key, so a later `GET` returns nil and `/api/token` answers 404. Redis stores **strings**, so the app converts models with `model_dump_json()` on write and `model_validate_json()` on read; `decode_responses=True` (`services/redis_client.py`) makes redis-py return `str` instead of `bytes`. Each single Redis command is atomic, but a *sequence* (read memory → modify → write memory) is not — that read-modify-write window is the lost-update risk discussed in [02a](02a-backend-and-data-technologies.md) §2a.4.

**Where used:** five Redis key families with explicit TTLs: `mockmind/backend/routers/prepare.py` (`setex` 14400 s), `mockmind/backend/services/report_store.py` (`STATUS_TTL`, `REPORT_TTL` = 86400), `mockmind/backend/services/memory_store.py` (1 year, SHA-256 key). Details: [02a](02a-backend-and-data-technologies.md) §2a.5 and [03](03-system-architecture.md) §3.6.

## 1.6.1 Concepts the task lists that this repo does **not** use
Transactions, indexes, migrations, connection pooling to a SQL DB, ORMs, cache invalidation protocols, background job queues (Celery/RQ), message brokers, embeddings, vector databases, RAG. **[Verified by search:** no such imports or usage in `mockmind/`**]**. They appear in [10](10-design-decisions-and-tradeoffs.md) only as *alternatives*. (Redis' client library does maintain a connection pool internally **[Standard knowledge]**, but the application doesn't configure it.)

## 1.7 Configuration, secrets and containers

- **Environment variables** configure programs without editing code; secrets go in a git-ignored `.env`, with a committed `.env.example` template (`mockmind/.env.example`).
- **`NEXT_PUBLIC_*`** variables in Next.js are inlined into browser code at **build time** — they are public by design.
- A **container** packages an app plus its dependencies; an **image** is the package; **Docker Compose** orchestrates several containers; services on a compose network reach each other by **service name** (`redis`, not `localhost`).
- A **health check** tells Docker when a service is really ready.

**`localhost` means different things.** Inside a container, `localhost` is *that container*. That is why `.env.example` has `REDIS_URL=redis://localhost:6379` (for running processes on your host) while `docker-compose.yml` overrides it with `redis://redis:6379` for the `backend` and `agent` containers (and `PISTON_URL=http://piston:2000`, `LANGFUSE_HOST=http://langfuse-web:3000`). Forgetting this is the classic "works on host, fails in Docker" bug. Meanwhile the *browser* always runs on your host, so `NEXT_PUBLIC_BACKEND_URL` stays `http://localhost:8000`.

**Required vs optional configuration [Verified in code]:** `GOOGLE_API_KEY` is read with `os.environ[...]` (crash with `KeyError` at first LLM call if missing); `LIVEKIT_*` likewise in `get_token`; `REDIS_URL`, `PISTON_URL`, `CORS_ORIGINS` have in-code defaults; Langfuse is skipped unless its keys are set.

**Where used:** `mockmind/docker-compose.yml`, three `Dockerfile`s, `mockmind/.env.example`. See [02e](02e-infra-and-tooling.md).

## 1.8 Authentication, tokens and trust

- **Authentication** = proving identity; **authorization** = what you may do.
- A **JWT** (JSON Web Token) is a signed token: header.payload.signature. Anyone can read the payload; nobody can alter it without the signing secret.
- **Capability URLs:** an unguessable URL *is* the credential.
**Anatomy of the token this app mints [Verified from `get_token`]:** the payload carries the identity (`candidate_<8 chars>`), a display name, an expiry of 2 hours (`with_ttl(timedelta(hours=2))`), a *video grant* (`room_join=True`, only for room `interview_{session_id}`, may publish audio, subscribe and publish data) and a *room configuration* with a `RoomAgentDispatch(agent_name="mockmind-interviewer", metadata={"session_id": ...})`. The signature uses `LIVEKIT_API_SECRET` (HMAC), which never leaves the backend. The browser can read all of this (base64) but cannot change it: any edit invalidates the signature and LiveKit rejects the connection. Notice what is *not* here: nothing authenticates the person calling `/api/token` — anyone who knows a valid `session_id` can request one. That is the "capability URL" trade-off.

**Where used:** LiveKit access tokens (`mockmind/backend/routers/session.py` → `get_token()`); there are **no user accounts** — sessions are protected only by random UUIDs. **[Verified]** Security analysis: [10](10-design-decisions-and-tradeoffs.md) and [07](07-engineering-challenges.md).

## 1.9 Testing and delivery basics

- **Unit test**: one function in isolation. **Integration test**: several parts together. **End-to-end (E2E)**: whole system like a user.
- **Fake / stub / mock**: stand-ins for dependencies (a dict pretending to be Redis; a canned LLM answer).
- **CI** runs tests automatically on each push. **Regression test**: protects a fixed bug.
- **Mutation check**: break the code on purpose to confirm a test fails.
**Example from this repo.** `backend/tests/conftest.py` defines `FakeRedis` (a dict with `get/setex/delete/ping/aclose`) and a `fake_redis` fixture that monkey-patches `redis_client` in four modules, so tests run in ~1 s with no server. **[Measured]:** `cd mockmind && backend/.venv/bin/python -m pytest backend/tests -q` → `53 passed, 1 skipped` on the analysis machine. Because LLM output is non-deterministic, those tests stub the LLM; judging *real* model behaviour is the job of the separate, paid, manual eval harness (`mockmind/evals/`, `.github/workflows/evals.yml`).

**Where used:** `mockmind/backend/tests/` (pytest), `.github/workflows/ci.yml`, `mockmind/evals/`. See [09](09-testing-evaluation-and-observability.md).

## 1.10 Machine learning and LLM fundamentals

- **Machine learning** learns patterns from data. A **large language model (LLM)** is a neural network trained to predict the next **token** (a chunk of text, ~¾ word). It generates text by repeatedly sampling the next token.
- The model has a **context window** (max tokens of input+output). Input and output tokens cost money and time.
- **Temperature** controls randomness (0 ≈ steadiest). **max output tokens** hard-caps the answer length; hitting it truncates mid-sentence.
- A **prompt** is the input text; a **system prompt** gives standing instructions.
- **Structured output / JSON mode** asks the model to emit JSON; it guarantees *syntax* (mostly), not your schema.
- **Hallucination**: fluent but false output.
- **Reasoning ("thinking") tokens**: some models (Gemini 2.5) can spend hidden tokens before the visible answer; they **count toward the output budget**.
- **Tool/function calling**: the model asks your code to run a function and consumes the result.
- **Agent vs workflow:** a *workflow* is steps you code; an *agent* lets the model choose the next step in a loop using tools.
- **Prompt injection**: instructions hidden in data that the model obeys.
- **Embeddings / RAG / fine-tuning** — **(not used here)**; see [02b](02b-ai-and-llm-deep-dive.md) §2b.9 for the verification that they are absent.

**Concrete settings in this repo [Verified]:** `services/code_judge.py` calls `gemini-2.5-flash` with `temperature=0.2`, `max_output_tokens=4096`, `response_mime_type="application/json"` and `thinking_budget=0` (no hidden reasoning, to keep the review fast and the whole 4096-token budget for visible JSON). If thinking were enabled, hidden tokens would eat into that budget and the JSON could be cut off mid-string — one reason structured output "guarantees syntax (mostly)" and the code still defends with `_clean_json()` (L61–67, strips ``` fences and extracts `{...}`) and, elsewhere, retries (`llm_retry.py`). Token counts and cost depend on provider pricing and are **[Not measured]** here.

**Where used:** everywhere in `mockmind/backend/agents/`, `mockmind/backend/services/report_generator.py`, `mockmind/agent/`. Deep dive: [02b](02b-ai-and-llm-deep-dive.md).

## 1.11 Audio, real-time and voice fundamentals

- **Digital audio**: the microphone's signal is *sampled* into numbers; a **codec** (Opus for speech) compresses it; audio is sent as small **packets**.
- **Latency** is delay; for conversation, replies slower than ~1–2 s feel broken **[Standard knowledge]**.
- **STT** (speech-to-text) transcribes; **TTS** (text-to-speech) synthesizes; **VAD** (voice activity detection) detects speech presence; **turn detection** decides the speaker finished; **barge-in** is interrupting the agent.
- **WebRTC**: browser real-time media; **ICE/STUN/TURN** connect through NAT/firewalls; an **SFU** (selective forwarding unit) relays streams (LiveKit Cloud is one).
- **Cascaded** voice agent = STT → LLM → TTS (what MockMind uses) vs **speech-to-speech** (one model).
**Where used:** `mockmind/agent/agent.py` → `entrypoint()`. Deep dive: [02c](02c-real-time-voice-and-livekit.md).

## 1.12 Reliability vocabulary used throughout

| Term | Meaning | Example in this repo |
|---|---|---|
| Retry with exponential backoff | try again, waiting longer each time | `mockmind/backend/services/llm_retry.py` → `retry_async()` |
| Timeout | stop waiting after a limit | 25 s agent-join timer (`interview/[sessionId]/page.tsx`); Piston 30 s (`code_executor.py`) |
| Graceful degradation | optional part fails, core continues | memory read/write wrapped in try/except (`prepare.py` → `_maybe_fetch_memory`) |
| Fail fast | stop loudly on essential failure | Redis `ping()` at startup (`main.py` → `lifespan`) |
| State machine | named states + transitions | report: `pending → ready | failed` (`report_pipeline.py`) |
| Checkpointing | save progress | transcript autosave (`session_state.py` → `_autosave`) |
| Idempotency | repeat is harmless | `retry_report()` is a no-op while `pending` |
| Never-raise boundary | a function swallows errors and reports state instead, so callers can't crash | `run_report_pipeline()` records `failed` and returns `False` (`report_pipeline.py` L11–54) |
| Defence in depth | multiple layers for one rule | follow-up cap: prompt + counter + deterministic eval check |

---

## 1.13 Self-check (short answers follow the list)
1. Why can't the browser use `EventSource` for `/api/prepare`?
2. What does `await` do, and what happens if you call a blocking function inside an `async def`?
3. Why is the backend–agent split mediated by Redis?
4. What does JSON mode guarantee and not guarantee?
5. What is the difference between a workflow and an agent?
6. What is a JWT and why can't the client forge one?

**Answers (short).**
1. `EventSource` can only send a `GET` with no body; `/api/prepare` must `POST` a multipart file, so the frontend reads the `fetch` response stream by hand (`hooks/usePreparation.ts`).
2. `await` suspends *this* coroutine and lets the event loop run others until the awaited I/O completes. A blocking call (e.g. synchronous `pypdf`, `time.sleep`) never yields, so the whole loop — all requests on that process — stalls until it returns.
3. They are separate processes (backend API container, LiveKit worker process per interview) that never call each other; Redis is the shared, expiring store both can reach (plan, state, report). Alternatives (direct RPC, queue) would add coupling or infrastructure; see [10](10-design-decisions-and-tradeoffs.md).
4. Guarantees (mostly) *valid JSON syntax*; does not guarantee that your *schema* (required fields, enum values, types) is respected, nor truthfulness. Hence Pydantic validation after parsing.
5. Workflow: code decides the sequence of LLM calls (preparation: resume → JD → questions). Agent: the model chooses actions via tool calls in a loop (the interviewer chooses `move_to_next_question` / follow-up). MockMind's interviewer is a *constrained* agent inside a code-controlled flow.
6. A JWT is `header.payload.signature`, signed with a secret known only to the server (here `LIVEKIT_API_SECRET`). The client can read the payload but cannot produce a valid signature after altering it.
