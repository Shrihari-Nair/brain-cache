# 02a — Backend and Data Technologies

> **Covers:** Python `asyncio`, FastAPI/ASGI/Uvicorn, Pydantic, Redis, Server-Sent Events, resume-parsing libraries, the `aiohttp` client.
> **Installed versions observed [Measured]** (from `mockmind/backend/.venv`): FastAPI 0.142.2, Starlette 1.7.0, Uvicorn 0.54.0, Pydantic 2.13.5, redis-py 8.1.0, pypdf 6.19.0, aiohttp 3.14.3, httpx 0.28.1. `requirements.txt` pins none of them (all `>=`): `fastapi>=0.110.0`, `uvicorn[standard]>=0.27.0`, `redis[asyncio]>=5.0.0`, `pydantic>=2.6.0`, `pypdf>=4.1.0`, `python-docx>=1.1.0`, `python-multipart>=0.0.9`, `aiohttp>=3.9.0`, `google-genai>=1.0.0`, `livekit-api>=0.7.0`, `langfuse>=3.0.0`, `python-dotenv>=1.0.0`. *Source: `mockmind/backend/requirements.txt`.* `python-multipart` is easy to overlook: FastAPI cannot parse `Form(...)`/`UploadFile` bodies without it, so `/api/prepare` depends on it even though no source file imports it.
>
> **Prerequisites for this chapter:** [01](01-fundamentals.md) §1.3 (HTTP), §1.5 (async), §1.6 (Redis/TTL).

---

## 2a.1 Python `asyncio` (cooperative concurrency)

### A. What is it?
A way to run many I/O-bound tasks *concurrently on one thread*. You write `async def` functions and `await` slow operations; an **event loop** switches between tasks whenever one awaits.

### B. Why does it exist?
Waiting dominates a backend's time: waiting for Gemini (seconds), Redis (milliseconds), clients. One thread per request wastes memory; blocking a single thread serves one request at a time. `asyncio` lets one thread serve many requests by interleaving them at `await` points.

### C. How does it work internally?
1. `async def f()` returns a **coroutine** object when called (nothing runs yet).
2. The **event loop** schedules coroutines as **tasks**.
3. When a task hits `await something_slow()`, it yields control; the loop runs another ready task; when the awaited I/O completes (OS notifies the loop), the task becomes ready again.
4. A task that never awaits (CPU work or a *blocking* call like a synchronous PDF parse) **monopolizes the loop**; nothing else on that loop runs.

| Responsibility split | |
|---|---|
| **asyncio/uvicorn do** | schedule tasks, notify on I/O completion |
| **Our code does** | mark I/O as `await`, avoid blocking calls, hold task references |

### D. Where it's used here
| Use | Source |
|---|---|
| All endpoints and services are `async def` | `mockmind/backend/routers/*.py`, `mockmind/backend/services/*.py` |
| Non-blocking backoff sleep | `mockmind/backend/services/llm_retry.py` → `retry_async()` (L10–33) |
| Background regeneration of reports | `mockmind/backend/routers/session.py` → `retry_report()` (L113–139) uses `asyncio.create_task` and a module-level `_retry_tasks` set (L109) |
| Coalesced transcript autosave | `mockmind/agent/session_state.py` → `_schedule_autosave()` / `_autosave()` (L93–110) |
| Fire-and-forget report start | `mockmind/agent/interviewer.py` → `InterviewerAgent.end_interview()` (L138–156; `asyncio.create_task` at L149) |
| Per-run timeout and bounded concurrency in evals | `mockmind/evals/run_eval.py` → `run_one()` (L59–161, `asyncio.wait_for`), `main()` (L172–222, `Semaphore`, `gather`) |
**[Verified]** by grepping every `create_task|gather|sleep|wait_for|Semaphore|to_thread` in `backend/`, `agent/`, `evals/`; there is **no** use of `asyncio.to_thread`, thread pools, or `run_in_executor`.

**Prerequisite: coroutine vs task vs "awaiting".** `await fetch()` runs the coroutine *inline* - the caller pauses until it finishes. `asyncio.create_task(fetch())` schedules it to run *alongside* the caller, which continues immediately. `asyncio.gather(a(), b())` starts both and waits for both. In this repo: `retry_report()` uses `create_task` (the HTTP request returns `202` right away while the report builds); `retry_async()` uses plain `await` plus `asyncio.sleep` (`llm_retry.py` L10-33: `attempts=3`, `base_delay=1.0`, delay = `base_delay * 2**i`, i.e. 1 s then 2 s, no sleep after the last attempt, re-raises the last error). Note `retry_async` catches *every* `Exception` as retryable (`# noqa: BLE001`), including programming errors; only the last is re-raised.

**Debounced autosave as a second example.** `AgentSessionState._schedule_autosave()` (`agent/session_state.py` L93-103) creates a task only if no autosave task is already pending; `_autosave()` (L105-110) sleeps `_AUTOSAVE_DELAY_S` (1.0 s, L15) then `save()`s. Several transcript entries within the delay window therefore cause one Redis write, not one per entry. A failed save is logged, not raised, so an autosave problem cannot kill the interview.

### E. Example (illustrative)
```python
async def handler():
    a = await fetch_profile()      # loop runs other tasks while waiting
    b = await fetch_job()          # sequential: waits for a first
    # concurrent version:
    a, b = await asyncio.gather(fetch_profile(), fetch_job())
```
**Actual code relationship:** `prepare_interview()` runs resume analysis and JD analysis **sequentially** (`prepare.py` L138–157) even though they are independent — `gather` would shorten preparation. **[Verified]**; the improvement is **[Inference]**.

### F. Alternatives
Threads (`concurrent.futures`) — simpler for blocking libraries but heavier; multiprocessing — true parallelism for CPU work; `trio`/`anyio` structured concurrency; Go/Node/Java ecosystems with different models.

### G. Trade-offs
+ Very efficient for I/O-bound services. − One blocking call stalls *every* request on that loop. − Exceptions in un-awaited tasks are easy to lose. − Debugging interleavings is harder.
**Concrete hazard in this repo [Verified]:** `extract_resume_text()` (synchronous `pypdf`/`python-docx` parsing) runs directly inside the async generator in `prepare_interview()` (L129). **[Hypothesis]** a large/complex 5 MB PDF could block the single backend event loop for noticeable time; **[Not measured]** (I did not time `pypdf` on large inputs). The cheapest mitigation would be `await asyncio.to_thread(extract_resume_text, ...)` **[Inference]**; not implemented. See [07](07-engineering-challenges.md) §7.3.

### H. Interview connection
"What is the event loop?" "What happens if you call `time.sleep` in an async function?" "Why keep a reference to a created task?" "Where would you use `gather` here?" → [11](11-interview-preparation.md).

---

## 2a.2 FastAPI, ASGI and Uvicorn

### A. What is it?
**FastAPI** is a Python web framework. **ASGI** (Asynchronous Server Gateway Interface) is the standard protocol between async Python web servers and apps. **Uvicorn** is an ASGI server that accepts network connections and calls the app.

### B. Why does it exist?
Web servers need a standard way to talk to application code (so frameworks and servers are interchangeable) and a convenient way to define routes, parse/validate inputs, and generate docs. FastAPI adds type-driven validation (via Pydantic) and automatic OpenAPI documentation on top of Starlette (the ASGI toolkit).

### C. How does it work internally?
1. Uvicorn receives bytes, parses HTTP, and calls `app(scope, receive, send)` (ASGI).
2. FastAPI/Starlette routes by method + path to your function, using the function signature to extract path params, query params, JSON bodies, form fields, files.
3. Request bodies are validated into Pydantic models (422 on failure).
4. Your function runs (on the event loop if `async def`; in a thread pool if plain `def`).
5. The return value is serialized to JSON, or a `StreamingResponse` iterates an async generator, sending each chunk as produced.
6. **Middleware** (e.g. CORS) wraps every request; **lifespan** code runs at startup/shutdown.

| Responsibility split | |
|---|---|
| **Framework does** | routing, parsing, validation, serialization, OpenAPI, streaming plumbing |
| **Our code does** | endpoint logic, error mapping (`HTTPException`), which data to stream |

### D. Where it's used here
- App creation, CORS, lifespan: `mockmind/backend/main.py` (lifespan L15-18; `FastAPI(title="MockMind API", version="1.0.0", lifespan=lifespan)`). `load_dotenv()` runs *before* the app and router imports so later `os.getenv` calls at import time (e.g. `PISTON_URL` in `code_executor.py`, `REDIS_URL` in `redis_client.py`) see `.env` values. Routers are registered for `health`, `prepare`, `session`, `memory`.
- **Startup/shutdown (`lifespan`)**: on start it `await redis_client.ping()` - if Redis is unreachable the app **fails to start** (fail fast); on shutdown `await redis_client.aclose()`.
- **CORS (`CORSMiddleware`)**: `allow_origins = CORS_ORIGINS.split(",")` (default `http://localhost:3000`; no whitespace stripping, so `"a, b"` would yield an origin `" b"` - **[Inference]** from the code, not run), `allow_credentials=True`, `allow_methods=["*"]`, `allow_headers=["*"]`.
- Endpoints: `GET /health` (`routers/health.py` L7-8), `POST /api/prepare` (`prepare.py` L96-225), `POST /api/token` (`session.py` L30-84), `GET /api/report/{session_id}` (L87-105), `POST /api/report/{session_id}/retry` (L112-139, `status_code=202`), `DELETE /api/candidate-memory` (`memory.py` L13-18, `status_code=204`).
- **Parameter binding in practice:** `prepare_interview()` declares `resume: UploadFile = File(...)`, `job_description: str = Form(...)` (both required, 422 if absent) and optional form fields `use_memory` (bool, default False), `candidate_email`, `interview_type` (`standard`/`coding`), `difficulty` (`easier|balanced|harder`), `length` (`quick|standard`), `focus_weak_areas`. Unknown preset values silently fall back to defaults (`resolve_plan_options`, L45-52). FastAPI infers `multipart/form-data` from the signature.
- **Error contract of `/api/report/{id}` [Verified]:** report key present -> 200 + JSON; status `failed` -> **502** with `detail={"status":"failed","message":...}`; anything else (still `pending`, never started, or expired) -> **404** "Report not ready yet." - the endpoint cannot distinguish "pending" from "unknown id". `/retry`: `pending` -> 202 `{"status":"pending"}` (no-op); report already exists -> 202 `{"status":"ready"}`; no `session_state` -> 404; empty transcript -> 409; otherwise sets `pending` and spawns `run_report_pipeline`.
- Running: Docker `CMD ["uvicorn","backend.main:app","--host","0.0.0.0","--port","8000"]` (`mockmind/backend/Dockerfile`); Compose overrides the command with `--reload` and bind-mounts the source (`mockmind/docker-compose.yml`, service `backend`). **Single process, no `--workers`** **[Verified]**. `--reload` (Compose only) makes Uvicorn watch files and restart the worker on change; it is a development feature.
- Auto-generated docs/specs: `GET /docs`, `GET /openapi.json`. **[Measured]**: `/openapi.json` lists 6 operations and 5 schemas (`Body_prepare_interview_api_prepare_post`, `ForgetMeRequest`, `TokenRequest`, `HTTPValidationError`, `ValidationError`).

### E. Example — the request validation that FastAPI performs for free (**[Measured]** with `curl` against the running backend)
```text
POST /api/token  {}   →  422  {"detail":[{"type":"missing","loc":["body","session_id"],"msg":"Field required","input":{}}]}
POST /api/token  {"session_id":"nope"} → 404 {"detail":"Session not found. Please run /api/prepare first."}
```
The 422 comes from the framework (because `TokenRequest.session_id` has no default); the 404 comes from our code (`session.py` L37–42).

### F. Alternatives
Flask (sync, simpler), Django/DRF (batteries included), Starlette (lower level), Litestar, Node/Express/Nest, Go/Gin. For a streaming + async + validated API, FastAPI is a natural fit. **[Standard knowledge]**

### G. Trade-offs
+ Productive, typed, async-native. − Response shapes of `prepare`/`token`/`report` aren't declared as `response_model`s, so OpenAPI documents inputs but not outputs **[Verified]**. − One process per container by default; CPU-bound work blocks the loop. − Three different error body shapes exist across endpoints (validation list, string `detail`, object `detail` for 502) **[Verified]**.

### H. Interview connection
"What is ASGI?" "Why does `/api/prepare` return 200 even when preparation fails?" "How would you scale this API?" "How does FastAPI validate a request?"

---

## 2a.3 Pydantic (data validation and serialization)

### A. What is it?
A library that defines data shapes as Python classes and validates/parses data into them at runtime.

### B. Why does it exist?
Python doesn't enforce types. External data (HTTP bodies, JSON from Redis, **LLM output**) is untrusted; you need a single place that rejects or coerces bad data.

### C. How does it work?
A `BaseModel` subclass with annotated fields builds a validator (Rust core in Pydantic v2). `Model(**data)` parses: it **coerces** compatible types in lax mode (`"4"`→`4`), **ignores unknown fields by default**, fills defaults, and raises `ValidationError` for mismatches. `model_dump_json()` / `model_validate_json()` convert to/from JSON strings. Enums validate membership.

| Responsibility split | |
|---|---|
| **Pydantic does** | validation, coercion, JSON (de)serialization |
| **Our code does** | model design, deciding which failures are retryable |

### D. Where it's used
All models in `mockmind/backend/models/interview_plan.py` (`QuestionCategory` L6–12, `InterviewQuestion` L33–43, `InterviewPlan` L67–83, `SessionState` L86–94) and `models/candidate_memory.py`; FastAPI request bodies (`TokenRequest`, `ForgetMeRequest`); `CodeReviewResult` (`code_judge.py`); the eval verdict `EvalVerdict` (`evals/judge.py`). Plans and states are stored in Redis as `model_dump_json()` strings.

### E. Example — **[Measured]** by running the real models:
```text
CandidateProfile(**{"name":"x","bogus":1})   → OK, "bogus" silently ignored
CandidateProfile(years_of_experience="lots") → ValidationError
CandidateProfile(years_of_experience="4")    → 4 (int)   # lax coercion
InterviewQuestion(category="coding", ...)   → ValidationError on 'category'
```
The last line is the exact failure that caused a real bug (see [07](07-engineering-challenges.md) §7.1, commit `b2ba44e`). **[Verified]** from `git show b2ba44e`: its message says the model sometimes emitted `'coding'` as the `category` instead of `question_type`, "crashing generate_questions() with an uncaught ValueError on roughly every other real call"; "every other call" is the author's own observation from manual testing, not measured in this analysis.

**Round trip used for Redis (illustrative; mirrors `prepare.py` L186-198 and `memory_store.py`):**
```python
plan_json = plan.model_dump_json()                      # str, written with SETEX
plan2 = InterviewPlan.model_validate_json(plan_json)    # what a reader does after GET
```
A model whose shape changed between writer and reader (e.g. a new required field) would make `model_validate_json` raise on *old* keys still alive in Redis (TTL up to 4 h / 24 h / 1 y) **[Inference]**; the memory model is the long-lived one (1-year TTL), so schema evolution matters most there.

### F. Alternatives
`dataclasses` (no validation), `attrs`+`cattrs`, `msgspec` (faster), TypedDict, JSON Schema validators, Marshmallow.

### G. Trade-offs
+ Strong boundary validation. − **Silently ignoring unknown fields** can hide LLM schema drift; − lax coercion may hide type mistakes; − a validation failure after an expensive LLM call wastes tokens (mitigated by retries).

### H. Interview connection
"Why validate LLM output with Pydantic?" "What does Pydantic do with extra fields?" "How does it serialize for Redis?"

---

## 2a.4 Redis (in-memory key–value store) and `redis.asyncio`

### A. What is it?
An in-memory data-structure server. Here it is used purely as **string key → JSON string value** storage with TTLs.

### B. Why does it exist?
Fast, simple shared state with built-in expiry, accessible by multiple processes.

### C. How does it work internally?
Redis keeps data in RAM and executes commands on a single thread, so each command is atomic. `SETEX key seconds value` stores a value and schedules deletion. Persistence (RDB snapshots / AOF log) is optional. The application's Redis container sets **no explicit persistence flags or named volume** in `docker-compose.yml`, so the `redis:7-alpine` image defaults apply. **[Measured]** on the running `mockmind-redis-1` container: `config get save` -> `3600 1 300 100 60 10000` (RDB snapshot after 1 change in 3600 s, 100 in 300 s, or 10000 in 60 s), `appendonly` -> `no`, and an *anonymous* Docker volume is mounted at `/data` (the official image declares `VOLUME /data`). So a plain container restart can retain data written before the last snapshot while recent writes can be lost, and `docker compose down -v` or recreating the container with a fresh anonymous volume discards it. This corrects the shorter "restart loses everything" wording that appears elsewhere in the series. The async client (`redis.asyncio`) maintains a **connection pool**; connections are bound to the event loop that created them.

| Responsibility split | |
|---|---|
| **Redis does** | store, expire, atomic single commands |
| **Our code does** | key naming, JSON (de)serialization, TTL choices, avoiding multi-key consistency assumptions |

### D. Where it's used
| Key | Writer → Reader | TTL | Source |
|---|---|---|---|
| `interview_plan:{id}` | backend prepare → backend token, agent | 14400 s | `prepare.py` L186–190; `session_state.py` `load()` |
| `session_state:{id}` | backend (empty) then **agent** → backend retry | 14400 s (refreshed on each save) | `prepare.py` L194–198; `session_state.py` `save()` (L112–134) |
| `interview_report:{id}` | report pipeline → backend GET | 86400 s | `report_pipeline.py` |
| `interview_report_status:{id}` | pipeline/retry → backend GET | 86400 s | `report_store.py` |
| `candidate_memory:{sha256(email)}` | pipeline → backend prepare | 31 536 000 s | `memory_store.py` L6–27 |
Client: `mockmind/backend/services/redis_client.py` (`redis.from_url(...)`, `decode_responses=True`). The agent also opens **short-lived** clients inside `AgentSessionState.load/save` (`session_state.py` L32–45, L112–134). **[Measured]** (re-run during audit): redis-py 8.1.0's pool for `redis.asyncio.from_url` reports `max_connections=100`. Note the agent's pattern: each `load()`/`save()` builds a brand-new client and `aclose()`s it in a `finally` - correct (no leak) but a new TCP connection per autosave **[Inference]**; the cost is **[Not measured]**.

### E. Example (illustrative commands you can run; see [13](13-glossary-and-reference.md))
```bash
docker compose exec redis redis-cli ttl interview_plan:<id>      # seconds remaining
docker compose exec redis redis-cli get interview_report_status:<id>
```

### F. Alternatives
PostgreSQL/DynamoDB (durable, queryable), Memcached (no persistence/structures), a message queue/stream for hand-offs (Redis Streams, SQS), in-process memory (not shareable across processes).

### G. Trade-offs
+ Very fast, minimal ops, TTL built in. − **No explicit persistence/named volume** in Compose; image-default RDB snapshots only (see §C, **[Measured]**) → recent writes can be lost on a crash and everything is lost if the volume is removed. − No cross-key transactions in our usage; memory is a read-modify-write (`get_memory` → merge → `save_memory`) with no lock → lost-update window **[Verified]** (`report_pipeline.py`, `memory_updater.py`). − Single instance; no `requirepass` on the app's Redis in dev compose (the Langfuse Redis *does* set `--requirepass`), and port 6379 is published to the host **[Verified]**.

### H. Interview connection
"Why Redis over Postgres here?" "What if Redis restarts?" "How do two processes share state safely?" "How do you avoid lost updates?"

---

## 2a.5 Server-Sent Events (server side)

### A. What is it? B. Why?
A browser-compatible, one-way streaming format over HTTP (`Content-Type: text/event-stream`); each message is `data: <text>` followed by a blank line. It avoids WebSocket complexity when only server→client updates are needed.

### C. How it works
The server keeps the HTTP response open and flushes chunks as events occur. Because HTTP status/headers are sent **before** the body, **errors mid-stream cannot change the status code** and must be conveyed as events.

### D. Used here
`mockmind/backend/routers/prepare.py` → `_sse()` (L228–229) formats; `prepare_interview()` returns `StreamingResponse(event_stream(), media_type="text/event-stream")` (L225); the generator is `event_stream()` (L117-223) and `_sse()` emits `f"data: {json.dumps(data)}\n\n"`. Stages and progress values: `parsing` 5→15, `resume_analysis` 20→45, `jd_analysis` 50→65, `question_gen` 70(→72)→90, `complete` 100, `error` 0 **[Verified]**.

**Failure paths [Verified from code]:** inside `event_stream()`, `PreparationError` (safe, user-facing: bad extension, empty file, oversize, empty extracted text, JD too long) is sent as `{"stage":"error","message": <that message>}`; any other exception is logged with the traceback and the client gets the generic `GENERIC_PREP_ERROR`, so internal details do not leak. In both cases the HTTP status is still `200`. The plan and an empty `session_state` are written to Redis only *after* question generation succeeds (L186-198), so a failed preparation leaves no plan key behind **[Inference]** from the code order. The `error` event still carries a `session_id`, but `/api/token` for it returns 404. If the client disconnects mid-stream the server will normally cancel the generator **[Standard knowledge]**; that path was not tested.

### E. Example (actual shape)
```text
data: {"stage": "parsing", "message": "Reading your resume...", "progress": 5, "session_id": "<uuid>"}

data: {"stage": "error", "message": "Unsupported file type. Please upload a PDF, DOCX or TXT resume.", "progress": 0, "session_id": "<uuid>"}
```
**[Measured]**: an unsupported-file request to the running backend produced exactly such an error event.

### F. Alternatives / G. Trade-offs
WebSockets (two-way), long-polling, polling, push via a queue. SSE is simple but one-way and ties up a connection per client during the whole preparation (20–60 s **[Not measured]**).

### H. Interview connection
"Why 200 on error?" "Why not `EventSource`?" (client side, [02d](02d-frontend-technologies.md)) "How would you make this survive a dropped connection?" (SSE here has no resume/ids — a gap.)

---

## 2a.6 Resume parsing and upload handling (`pypdf`, `python-docx`)

### A–B. What/Why
Libraries that extract text from PDF and DOCX so the text can be sent to an LLM.

### C. How
`pypdf` parses the PDF object structure and extracts text from each page's content stream; **scanned PDFs have no text layer** and yield nothing (OCR would be needed). `python-docx` reads paragraphs from the DOCX XML. `.txt` is decoded as UTF-8 with errors ignored. `python-docx` reads the modern `.docx` format (a zip of XML); it does **not** read legacy binary `.doc` **[Standard knowledge]**, yet `.doc` is in `ALLOWED_RESUME_EXTS` and routed to the DOCX reader (`resume_parser.py` L22-29), so a true `.doc` file would presumably raise inside the parser and surface as the generic error - **[Hypothesis]**, not tested. `extract_text_from_docx` reads only body paragraphs (`doc.paragraphs`), so text inside tables is skipped **[Inference]** from the code.

### D. Used here
`mockmind/backend/services/resume_parser.py` → `extract_text_from_pdf()` (L5–13), `extract_text_from_docx()` (L16–19), `extract_resume_text()` (L22–29, dispatch by extension). Guards in `prepare.py` (L56-59): `MAX_RESUME_BYTES` = 5 MB, `ALLOWED_RESUME_EXTS` = `{.pdf,.docx,.doc,.txt}`, `MAX_RESUME_CHARS` = 30 000 (text is *truncated*, not rejected), `MAX_JD_CHARS` = 15 000 (rejected if longer); `await resume.read(MAX_RESUME_BYTES + 1)` (L127) reads one byte past the cap so oversize is detected without buffering an unbounded upload; `validate_resume_upload()` (L69–78).

### E. Example (the oversize check, simplified/illustrative)
```python
data = await upload.read(LIMIT + 1)
if len(data) > LIMIT: raise PreparationError("too large")   # never loaded more than LIMIT+1 bytes
```

**Limit of this guard [Inference]:** Starlette parses the multipart body (spooling it to a temporary file) *before* the endpoint runs, so `read(LIMIT+1)` bounds what the *handler* loads into memory, not what the server already received; a reverse proxy would be the place to cap raw upload size. I did not read Starlette's multipart code in this audit. `validate_resume_upload()` runs *after* the read, and the empty-text check ("Could not extract text from resume...") is what catches scanned PDFs.

### F. Alternatives / G. Trade-offs
PyMuPDF/pdfminer (often faster/better layout), OCR (Tesseract/cloud) for scans; Unstructured. Current limits: layout-heavy resumes lose structure; **extension, not content type, decides the parser** (no magic-byte check) **[Verified]**; parsing is synchronous (see §2a.1 G).

### H. Interview connection
"How do you protect against huge uploads?" "What if the PDF is a scan?" "Why truncate to 30 000 characters?"

---

## 2a.7 `aiohttp` (async HTTP client)

### A–C. What/Why/How
An async HTTP client library; each `await session.post(...)` yields to the event loop while waiting. (Its only production caller path runs inside the **agent** process, because only the agent calls the code-execution service.)

### D. Used here
`mockmind/backend/services/code_executor.py` → `execute_code()` (L49–78) and `_ensure_language_version()` (L17–46) call the Piston sandbox: `GET /api/v2/runtimes`, `GET/POST /api/v2/packages`, `POST /api/v2/execute`, with `ClientTimeout(total=30)` and `run_timeout` of 3000 ms; base URL from `PISTON_URL` (default `http://localhost:2000`, read once at import); also `mockmind/evals/preflight.py`.

**Failure paths [Verified]:** `resp.raise_for_status()` turns any non-2xx into `aiohttp.ClientResponseError`; connection problems/timeouts raise `aiohttp`/`asyncio.TimeoutError`; an unknown language raises `ValueError`. `review_code()` (`code_judge.py` L70-136) wraps each test run in `try/except Exception` and records `passed: False, stderr: "execution error: ..."`, so a dead sandbox yields failed tests rather than a crash. A new `ClientSession` is created per `execute_code` call (fine here; a shared session would reuse connections). The script file is always named `main.py` for Python and `main` (no extension) for other languages; whether Piston accepts that for every language is **[Not verified]**.
**Code smell [Verified]:** `execute_code` contains two consecutive string literals; the second one (the "Returns {...}" text) is a no-op expression, not part of the docstring.

### F–G. Alternatives / trade-offs
`httpx` (also installed and used in tests) offers a similar async API; having both `aiohttp` and `httpx` is a minor redundancy **[Verified]** (httpx is not in `requirements.txt`; it is installed ad hoc for tests, see [02e](02e-infra-and-tooling.md) §2e.5).

---

## 2a.8 Interview checklist for this chapter
- Explain event loop vs threads vs processes with the actual repo hazards (`extract_resume_text` blocking; `silero.VAD.load` blocking — see [02c](02c-real-time-voice-and-livekit.md)).
- Explain why plans/states/reports are JSON blobs in Redis and what that costs.
- Explain SSE vs polling vs WebSocket using `/api/prepare` and `/api/report`.
- Explain Pydantic's role as the AI-output boundary and its failure modes.
