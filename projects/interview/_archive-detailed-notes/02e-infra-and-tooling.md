# 02e — Infrastructure and Tooling

> **Scope:** Docker and Compose, configuration/secrets, the Piston code sandbox, Langfuse and OpenTelemetry, pytest, GitHub Actions, package tooling.
> **Prerequisites:** [01](01-fundamentals.md) §1.7 (config, containers) and §1.9 (testing/CI).
> **Observed environment [Measured]** (re-confirmed during audit): Docker 29.4.3; Python 3.12.3 (venvs); Node v24.21.0 on the host. `docker compose ps` showed `backend`, `redis`, `piston` and all `langfuse-*` services running; the `agent` and `frontend` containers were **not** running (this is the state of the analysis machine, not a defect).

---

## 2e.1 Docker and Docker Compose

### A. What is it?
**Docker** packages an application and its dependencies into an **image**; a running image is a **container**. **Compose** describes a multi-container system in `docker-compose.yml`.

### B. Why does it exist?
"Works on my machine" problems: different OS/library versions. Containers make environments reproducible and let many services start together.

### C. How it works
Images are built from a **Dockerfile** as cached layers (copying `requirements.txt` before the code makes dependency installation a cached layer). Containers share the host kernel but are isolated by namespaces/cgroups. On a Compose network, services resolve each other by **service name**. `depends_on: condition: service_healthy` waits for a health check. Volumes persist data; bind mounts map host folders in. A **named volume** is managed by Docker and survives `docker compose down` (but not `down -v`); an **anonymous volume** is created implicitly when an image declares `VOLUME` (the official Redis image does, for `/data`) and is orphaned when the container is recreated. A **build context** is the folder sent to the Docker daemon; `COPY . .` copies *everything in the context* unless a `.dockerignore` excludes it. A **multi-stage build** uses several `FROM` stages so the final image can omit build tools. **Layer caching** means Docker reuses a layer when its inputs are unchanged, so order matters: dependency files first, code last.

### D. How it's used here **[Verified]** (`mockmind/docker-compose.yml`, `docker compose config -q` exits 0)

| Service | Image / build | Host ports | Notes |
|---|---|---|---|
| `redis` | `redis:7-alpine` | 6379 | health check `redis-cli ping` (every 5 s, 3 s timeout, 5 retries); no named volume and no persistence flags (see 02a §2a.4 for **[Measured]** image defaults: RDB `save 3600 1 300 100 60 10000`, AOF off, anonymous `/data` volume) |
| `piston` | `ghcr.io/engineer-man/piston` (**no tag**, i.e. `latest`) | 2000 | `privileged: true`, named volume `piston_packages:/piston/packages` (installed language runtimes persist), `tmpfs /tmp:exec`, `restart: unless-stopped` |
| `backend` | build `backend/Dockerfile` | 8000 | `env_file: .env`; `REDIS_URL=redis://redis:6379`, `PISTON_URL=http://piston:2000`, `LANGFUSE_HOST=http://langfuse-web:3000`; bind-mounts `.:/app`; command `uvicorn backend.main:app ... --reload`; waits for healthy Redis |
| `agent` | build `agent/Dockerfile` | — | same env; command `python agent/agent.py start`; bind-mount |
| `frontend` | build `frontend/Dockerfile` (context `./frontend`) | **3002→3000** | build args `NEXT_PUBLIC_LIVEKIT_URL` (from `${LIVEKIT_URL}`), `NEXT_PUBLIC_BACKEND_URL=http://localhost:8000`; `depends_on: backend` (start order only, no health condition) |
| `langfuse-postgres` / `-clickhouse` / `-minio` / `-redis` / `-worker` / `-web` | `postgres:17`, `clickhouse/clickhouse-server:25.12`, `cgr.dev/chainguard/minio`, `redis:7-alpine` (with `--requirepass` and `--maxmemory-policy noeviction`), `langfuse-worker:4`, `langfuse:4` | 9090→9000 (MinIO API), 3001→3000 (Langfuse web); MinIO console 9001 and ClickHouse are *not* published | named volumes `langfuse_postgres_data`, `langfuse_clickhouse_data`, `langfuse_clickhouse_logs`, `langfuse_minio_data`, `langfuse_redis_data`; secrets via `${LANGFUSE_*}`; `langfuse-web`/`-worker` wait (via a YAML anchor `*langfuse-depends-on`) for all four stores to be healthy; `TELEMETRY_ENABLED: "false"`; `langfuse-web` pre-creates org `mockmind-org`, project `mockmind` and an admin user via `LANGFUSE_INIT_*` |
**No literal secrets are present in the compose file**; every secret is a `${VARIABLE}` reference **[Verified by grep]**. (Non-secret literals such as MinIO user `minio`, ClickHouse user `clickhouse`, org/project ids are in the file.) Compose substitutes `${VAR}` from the shell environment or a `.env` file next to the compose file; note this is a *different mechanism* from `env_file: .env`, which injects variables into the container. The top-level `version: "3.9"` key is obsolete: **[Measured]** `docker compose config -q` exits 0 but prints a warning that the attribute is ignored.

Dockerfiles: `backend/Dockerfile` and `agent/Dockerfile` are `python:3.12-slim`, install requirements, `COPY . .`; the backend CMD is `uvicorn backend.main:app --host 0.0.0.0 --port 8000`. `frontend/Dockerfile` is a **multi-stage** build on `node:18-alpine` (stages `deps` → `builder` → `runner`): `npm ci`, then `npm run build` with the `NEXT_PUBLIC_*` build args exported as `ENV` (that is *why* they must be build args: Next.js inlines them into the browser bundle at build time), then a runner stage that copies `.next`, `public`, `node_modules`, `package.json` and runs `npm run start` on port 3000. No Dockerfile sets a `USER`, so all three run as root **[Verified]**.

**Secrets-in-image finding [Measured]:** there is **no `.dockerignore`** in `mockmind/` or `mockmind/frontend/`, and the backend/agent images are built with context `mockmind/` and `COPY . .`. I listed `/app` inside the existing `mockmind-backend` image (names only, no contents): it contains `.env` (alongside `.env.example`, `evals/`, `frontend/` and a 627 MB `frontend/node_modules`); the image is 2.25 GB. So any real keys in a local `.env` are baked into the image layers at build time. In Compose development the bind mount `.:/app` shadows that copy, but anyone who pushed or shared such an image would be sharing the keys. This is a **local-build hygiene issue, not a repository leak** (`.env` is git-ignored: `git check-ignore -v .env` → `mockmind/.gitignore:17`). Fix would be a `.dockerignore` (not applied: this audit does not modify config).

### E. Example (the dev/prod difference in one line)
Compose runs the backend with the **source bind-mounted and `--reload`** (development ergonomics); the Dockerfile's `COPY . .` + plain `uvicorn` is what a built image runs. **[Verified]**

### F. Alternatives
Kubernetes/ECS for orchestration; Podman; systemd units; serverless (not suited for long-lived voice workers).

### G. Trade-offs
+ One-command local stack. − The dev setup mounts source into production-like containers; Redis has no explicit persistence config; Piston is privileged and unpinned (`latest`); **Node 18 in the image vs Node 20 in CI**; Python dependencies unpinned; no `.dockerignore`; containers run as root; the Compose frontend is published on host port 3002 while README and the backend CORS default assume 3000 (so `CORS_ORIGINS` must include `http://localhost:3002` for that container — **[Inference]**, not run). No resource limits or `restart` policy for `backend`/`agent`/`frontend`; the `agent` has no health check. **[Verified]** from the files.

### H. Interview connection
"How do containers find each other?" "Why `depends_on` with health checks?" "What's the risk of a privileged container?" "How would you deploy this?"

---

## 2e.2 Configuration and secrets

**What exists [Verified]:** `mockmind/.env.example` (template), a git-ignored `mockmind/.env` (not read during this audit), `mockmind/frontend/.env.local.example` (names: `NEXT_PUBLIC_LIVEKIT_URL`, `NEXT_PUBLIC_BACKEND_URL`). Variables: `LIVEKIT_URL`, `LIVEKIT_API_KEY`, `LIVEKIT_API_SECRET`, `GOOGLE_API_KEY`, `CARTESIA_API_KEY`, `SONIOX_API_KEY`, `REDIS_URL`, `PISTON_URL`, `CORS_ORIGINS`, `LANGFUSE_HOST`, `LANGFUSE_PUBLIC_KEY`, `LANGFUSE_SECRET_KEY`, plus infra secrets for the Langfuse containers.
**Who reads what [Verified by grep of `os.environ`/`os.getenv`]:**

| Variable | Required? | Read by |
|---|---|---|
| `GOOGLE_API_KEY` | yes, `os.environ[...]` at first LLM call | every `genai.Client(...)` call site in `backend/agents/*`, `services/report_generator.py`, `code_judge.py`, `memory_updater.py`, `evals/*` |
| `LIVEKIT_URL`, `LIVEKIT_API_KEY`, `LIVEKIT_API_SECRET` | yes | backend `get_token()` (`session.py` L44–46); the LiveKit worker reads the same names itself **[Library source]** `livekit/agents/worker.py` L334–336 |
| `CARTESIA_API_KEY`, `SONIOX_API_KEY` | yes for voice | read by the plugins, not by repo code (**[Library source]**: Cartesia TTS falls back to `os.environ.get("CARTESIA_API_KEY")`; the Soniox one was not opened) |
| `REDIS_URL` | default `redis://localhost:6379` | `redis_client.py`, `agent/session_state.py` (twice) |
| `PISTON_URL` | default `http://localhost:2000` | `code_executor.py`, `evals/preflight.py` |
| `CORS_ORIGINS` | default `http://localhost:3000` | `backend/main.py` |
| `LANGFUSE_PUBLIC_KEY`, `LANGFUSE_SECRET_KEY` | optional | `tracing.is_enabled()` (both must be set); the `langfuse` SDK `get_client()` also reads `LANGFUSE_HOST` itself **[Standard knowledge]** |
| `LANGFUSE_HOST` | optional | `agent._setup_otel_tracing()` requires host **and** both keys, otherwise logs "Langfuse not configured" |
| `LANGFUSE_SALT`, `_ENCRYPTION_KEY`, `_NEXTAUTH_SECRET`, `_POSTGRES_PASSWORD`, `_CLICKHOUSE_PASSWORD`, `_REDIS_AUTH`, `_MINIO_ROOT_PASSWORD`, `_INIT_USER_EMAIL`, `_INIT_USER_PASSWORD` | only for the Langfuse containers | `docker-compose.yml` substitution; `.env.example` placeholders are `change-me`-style, and its comments say to generate with `openssl rand -hex 32` (encryption key) / `-hex 16` (others) |

Because `.env.example` sets `REDIS_URL=redis://localhost:6379`, but Compose overrides `REDIS_URL`, `PISTON_URL` and `LANGFUSE_HOST` for `backend`/`agent` via `environment:` (which takes precedence over `env_file`), the same `.env` serves both host-run and container-run processes.

**Loading:** `python-dotenv` (`load_dotenv()` at the top of `backend/main.py` and `agent/agent.py`) and Compose `env_file`. Code reads required secrets with `os.environ[...]` (raises `KeyError` if missing) and optional ones with `os.getenv(...)`.
**Behavioral switches by presence [Verified]:** backend Langfuse tracing turns on only if public+secret keys are set (`tracing.py` `is_enabled()` L9–10); the agent's OTel export additionally requires `LANGFUSE_HOST` (`agent.py` `_setup_otel_tracing()` L73–113). So it is possible to have backend generations traced but the agent silently not traced if `LANGFUSE_HOST` is unset (the Langfuse SDK has its own default host **[Standard knowledge]**; not tested).
**Risk:** if LiveKit variables are missing, `get_token()` raises `KeyError` → HTTP 500 (not handled); a missing `GOOGLE_API_KEY` surfaces during preparation as the generic SSE error message (the `KeyError` is caught by the broad `except Exception` in `event_stream()`). **[Verified from code]**; neither was provoked. **Secret hygiene:** this series records variable *names* only; `.env` contents were not read. See also the image-baking note in §2e.1.

---

## 2e.3 Piston (sandboxed code execution)

### A–B. What/why
An open-source service that runs untrusted code in isolated sandboxes with resource and time limits, exposed over HTTP. Needed because executing candidate code in the backend would let it read secrets, exhaust resources or damage the host.

### C. How it works **[Standard knowledge + usage verified]**
Each request creates an isolated environment (Linux namespaces/cgroups), writes files, runs the language runtime with CPU/memory/time limits, captures stdout/stderr/exit code, then discards the environment. Languages are installed as packages (our code installs Python on first use if absent).

### D. Used here **[Verified]**
`mockmind/backend/services/code_executor.py`:
- `_ensure_language_version()` (L17–46): `GET /api/v2/runtimes`; if missing, `GET /api/v2/packages` and `POST /api/v2/packages` to install the latest version; caches versions in `_version_cache`.
- `execute_code()` (L49–78): `POST /api/v2/execute` with `files=[{name, content}]`, `stdin`, `run_timeout` (3000 ms default), total HTTP timeout 30 s; returns `stdout`, `stderr`, `exit_code`, `signal`, `compile`.
`mockmind/backend/services/code_judge.py` → `review_code()` (L70–136), imported and called by the **agent process** (`agent.py` L151–152, inside `_handle_code_submission` L130–159) and the eval driver (not by any backend router), runs each test case as `submitted_code + "\n\n" + test.input` and compares stripped stdout to `expected_output`, requiring `exit_code == 0`. Ground truth wins: after the LLM review, `tests_passed`/`tests_total` are overwritten from the real results (L132–133). Each test is a separate Piston request (so N test cases = N sandbox runs) **[Verified]**; their latency is **[Not measured]**.

**Concrete example.** For a coding question whose spec has test input `print(add(2,3))` and expected `5`, the program sent to Piston is the candidate's code followed by `print(add(2,3))`; stdout `"5\n"` is stripped to `"5"` and matches. A candidate whose code runs forever is stopped by Piston's `run_timeout` (3000 ms), reported through `signal`/`stderr`. If Piston is down, the test is recorded as `execution error: ...` and the LLM judge is told so via `test_results_json`; if the later review call itself fails, the agent speaks `REVIEW_FAILED_INSTRUCTIONS` (`agent.py` L151–156).
**Security caveat [Verified]:** the container is `privileged: true` (Piston builds nested sandboxes). Acceptable locally; a risk if exposed.

### F–G. Alternatives/trade-offs
Firecracker/gVisor sandboxes, cloud code-execution APIs (Judge0), WASM runtimes; running code in the backend (unsafe). Trade-offs: privileged container; first run installs the language (slower; `_version_cache` is a per-process dict, so each new process re-queries `/runtimes`); unknown concurrency limits; the docstring in `code_executor.py` notes the 3000 ms cap matches Piston's server default (`PISTON_RUN_LIMIT` configures it) — **[Verified]** as a code comment, Piston's own defaults not checked. **[Not measured]**

---

## 2e.4 Observability: Langfuse and OpenTelemetry

### A. What is it?
**Tracing** records the path and timing of operations. **OpenTelemetry (OTel)** is a vendor-neutral standard (spans, exporters; **OTLP** is its wire protocol). **Langfuse** is an open-source LLM observability platform (generations, sessions, scores), self-hosted here.

### D. How it's wired **[Verified]**
- **Backend/evals:** `traced_generation(name, session_id=..., input_data=..., model=...)` (`mockmind/backend/services/tracing.py` L44–72) is a context manager. If Langfuse keys are absent it yields a no-op handle (`_NoopGenerationHandle`, L35–40). Otherwise it starts a Langfuse "generation" under `propagate_attributes(session_id=...)`; `gen.record_response(response)` (L17–29) stores the output and token usage (`prompt_token_count`, `candidates_token_count`, `total_token_count`); an exception in the block is recorded as an ERROR and re-raised.
- **Agent:** `_setup_otel_tracing()` (`agent.py` L73–113) builds a `TracerProvider` with a `SessionSpanProcessor` (stamps `session.id`, read from the job metadata) and a `BatchSpanProcessor(OTLPSpanExporter(endpoint=f"{host}/api/public/otel", headers=Basic auth))`, registers it with LiveKit's `set_tracer_provider`, and a shutdown callback flushes (`agent.py` L259–263; `_trace_provider.force_flush()`). Each export carries header `x-langfuse-ingestion-version: 4` and Basic auth built from the public/secret key pair (the secret is never logged by this code).
- **Eval scores:** `run_one()` pushes each boolean verdict/check as a Langfuse score `eval_<name>` attached to the session (`mockmind/evals/run_eval.py`).
**Correlation:** the same `session_id` is used as Langfuse session id across backend generations and agent spans, so one interview is one session (see [03](03-system-architecture.md) §3.9).

### Prerequisite: what a span, a trace and a generation are
A **span** is one timed operation with attributes (name, start/end, parent). A **trace** is a tree of spans for one request/session. A Langfuse **generation** is a span specialised for an LLM call (model, input, output, token usage). `session.id` is just an attribute that Langfuse uses to group traces. In this repo the backend creates generations directly with the Langfuse SDK, while the agent emits OTel spans that LiveKit Agents already produces for STT/LLM/TTS turns and tool calls (**[Library source]**-based claim for LiveKit's span emission; only the registration is visible in repo code).

**Example.** A candidate runs prepare → interview → report. In Langfuse you would see one session (the `session_id`) containing backend generations such as `resume_analysis`, `jd_analysis`, question generation, `code_review`, report generation, plus agent spans; on any single generation you can read the prompt, the response and `prompt_token_count`/`candidates_token_count`/`total_token_count` (recorded by `record_response`, `tracing.py` L17–29). I did not open the Langfuse UI in this audit, so what the UI displays is **[Not verified]**.

### G. Limits **[Verified]**
Prompts (including resume text) and responses are stored by Langfuse in its own databases; no retention/redaction is configured. No metrics/alerts exist. An earlier agent log showed OTLP export 404s (non-fatal; configuration mismatch) — **observed in a previous session, not re-verified in this analysis**.

---

## 2e.5 pytest and the test tooling

`pytest` runs `test_*.py` files. The repo's suite lives in `mockmind/backend/tests/` with shared fixtures in `conftest.py` (`FakeRedis`, `fake_redis`, `make_plan`). `httpx` is used for `TestClient`/`ASGITransport`; `PyJWT` decodes the LiveKit token in the smoke test (it arrives transitively with `livekit-api`). pytest and httpx are **not** in `requirements.txt` — they're installed ad hoc locally and explicitly in CI. **[Verified]** There is no `pytest.ini`, `pyproject.toml` or `setup.cfg`, so pytest uses defaults and finds `mockmind/backend/tests/test_*.py` when run from `mockmind/` (`python -m pytest backend/tests -q`; `python -m` puts the current directory on `sys.path`, which is what makes `import backend...` work). The folder has 15 `test_*.py` files plus `conftest.py` and `__init__.py`, and two script-style `.sh` files (`test_memory_endpoint.sh`, `test_prepare_memory.sh`) that pytest does **not** collect. **[Measured]** `53 passed, 1 skipped` in about 1.4 s (re-run during audit). `conftest.py::fake_redis` monkey-patches `redis_client` in four modules; `test_api_smoke.py` is the test that uses a real Redis (the CI service) according to the workflow comment — it is guarded by `pytestmark = pytest.mark.skipif(not _redis_up(), ...)` (L48), so without a reachable Redis the module is skipped rather than failed. Which test produced the one skip in the **[Measured]** run was not checked (pytest `-rs` would show it). More in [09](09-testing-evaluation-and-observability.md). Details and verified runs: [09](09-testing-evaluation-and-observability.md).

---

## 2e.6 GitHub Actions (CI)

`.github/workflows/ci.yml` (**[Verified]**): triggers on push to `main` and on pull requests; `concurrency` cancels superseded runs; `permissions: contents: read`.
- **backend job** (working dir `mockmind`; env `REDIS_URL=redis://localhost:6379`; no API keys, the LLM is always stubbed): `actions/checkout@v4`, `actions/setup-python@v5` with pip cache keyed on `mockmind/backend/requirements.txt`, Ubuntu, Python 3.12, a **Redis service container** with a health check (`redis-cli ping`, 5 s interval, 10 retries), `pip install -r backend/requirements.txt`, then `pip install "livekit-agents~=1.5" pytest httpx`, then `python -m pytest backend/tests -q`.
- **frontend job:** Node 20 (the Docker image uses Node 18), `npm ci`, `npx tsc --noEmit`, `npm run build` (dummy `NEXT_PUBLIC_BACKEND_URL=http://localhost:8000`, `NEXT_PUBLIC_LIVEKIT_URL=wss://example.livekit.cloud`). There is no lint step (`npm run lint` exists in `package.json` but CI does not call it).
- **Not in CI:** agent plugin dependencies (STT/TTS/VAD — only core `livekit-agents~=1.5` is installed, per the workflow comment), Docker image builds, the evals, any security/dependency scanning.
`.github/workflows/evals.yml`: **manual** (`workflow_dispatch`) LLM evals. Inputs: `persona` (choice: `all, coding, strong, vague, prompt_injector, coder, buggy_coder`, default `all`), `runs` (default `3`), `min_pass_rate` (default `0.8`). A Piston service container runs `--privileged --tmpfs /tmp:exec` on port 2000; `PISTON_URL=http://localhost:2000`; job `timeout-minutes: 60`. The first step fails fast with an error annotation if the `GOOGLE_API_KEY` secret is empty. Inputs are routed through env vars (`PERSONA`, `RUNS`, `MIN_PASS_RATE`) and used as quoted shell variables, which avoids script injection from `${{ inputs.* }}` interpolation. The command is `python evals/run_eval.py --persona "$PERSONA" --runs "$RUNS" --concurrency 3 --min-pass-rate "$MIN_PASS_RATE"`; `evals/results/` is uploaded with `if: always()` as artifact `eval-results`. Only `backend/requirements.txt` is installed (not the agent requirements).
**Not verified:** neither workflow has been executed on GitHub as part of this analysis; I ran the same commands locally (see [14](14-coverage-report-and-open-questions.md)).

---

## 2e.7 Package tooling
Python: `uv` creates venvs and installs (README lines ~168–174: `uv venv backend/.venv`, `uv pip install -r backend/requirements.txt --python backend/.venv/bin/python`, and the same for `agent/.venv`); Docker and CI use plain `pip`. Node: `npm` with `package-lock.json` (CI/Docker use `npm ci`, which installs exactly the lockfile). No lockfile for Python, so builds are not reproducible across dates **[Verified]**; e.g. the agent venv has LiveKit Agents 1.8.4 although the requirement is only `~=1.5` (**[Measured]**). Two venvs exist (`backend/.venv`, `agent/.venv`) because the agent needs the heavy plugin stack. Docker images are large (**[Measured]** backend image 2.25 GB, mostly because of the whole-folder `COPY`, see §2e.1; the split between layers was not analysed).

---

## 2e.8 Interview connection
- Explain the compose network, health checks, and why Redis has no volume.
- Explain how tracing is made optional and how sessions correlate across processes.
- Explain Piston's isolation and its privileged-container risk.
- Explain what CI does and what it can't tell you (LLM behaviour).
