# 21 - Deployment and infrastructure (tag: DEP)

## What this file covers

How AEOS is packaged and run: the `Dockerfile`, the `docker-compose.yml` topology (default stack plus `observability` and `a2a` profiles), configuration and secrets (`core/config.py`, `.env`, `scripts/init_env.py`, `scripts/check_no_secrets.py`), startup ordering and health checks, the CI workflow, the Makefile and uv/ruff/mypy/pytest tooling, the web image, and an honest list of what is missing for production. Observability pieces are in [20-observability-and-telemetry](20-observability-and-telemetry.md); the API and auth behind the ports are in [18-api-and-client](18-api-and-client.md); sandbox/Docker-socket implications are in [12-sandbox-and-execution-safety](12-sandbox-and-execution-safety.md). Labels: **Verified**, **Inferred**, **Potential failure scenario**, **Recommendation**. No secret values were read or are quoted here (`.env` was not opened; only `.env.example` key names were listed).

---

## 1. One image, many commands (Dockerfile)

Concept: if every service is the same codebase, build one image and vary the command. It guarantees the API, worker and each A2A specialist run identical code.

Verified (`Dockerfile`):
- Base `python:3.12-slim`; installs `git` (agents and the demo repo use git) and nothing else via apt.
- `uv` is copied from `ghcr.io/astral-sh/uv:0.11` (pinned minor).
- Layer caching trick: copy `pyproject.toml` + `uv.lock`, run `uv sync --frozen --no-dev --no-install-project` (dependencies only), *then* copy `src/`, `policy/`, `demo_target/` and sync again to install the project. Editing source does not reinstall dependencies.
- `--frozen` means the lockfile is authoritative (fails rather than re-resolving). `UV_COMPILE_BYTECODE=1`, `PYTHONUNBUFFERED=1`, venv on `PATH`.
- `EXPOSE 8000 9101 9102 9103` (API plus three A2A specialists); default `CMD ["aeos", "--help"]` - compose always overrides the command.
- Not present (Verified by reading): no `USER` directive (runs as root), no `HEALTHCHECK` (compose defines one), no multi-stage build for Python, `.git`-less image.

The web image (`web/Dockerfile`) is a two-stage `node:20-alpine` build: `npm ci`, build with `NEXT_PUBLIC_AEOS_API` inlined, then a runtime stage copying `.next` and `node_modules`, `CMD npx next start -p 3100`. Note it ships all `node_modules` (not Next's `standalone` output), so the image is larger than necessary (Inferred).

## 2. Compose topology

```mermaid
flowchart TB
  subgraph default[Default stack: docker compose up -d]
    PG[(postgres pgvector:pg16 :5432)]
    RD[(redis :6380)]
    OC[otel-collector :4317/4318/13133]
    SEED[aeos-seed one-shot]
    API[api: aeos serve + embedded worker :8800->8000]
    WEB[web Next.js :3100]
  end
  subgraph obs[profile observability]
    LW[langfuse-web :3000]
    LWK[langfuse-worker]
    CH[(clickhouse)]
    MN[(minio)]
    LR[(langfuse-redis)]
    LDB[(langfuse-db pg17)]
  end
  subgraph a2a[profile a2a]
    AN[analytics :9101]
    RS[research :9102]
    CD[coding :9103]
    API2[api-a2a :8801->8000]
  end
  SEED -->|needs healthy| PG
  API -->|needs completed| SEED
  WEB -->|needs healthy| API
  API -. OTLP .-> OC
  OC -. OTLP/HTTP .-> LW
  LW --> LWK
  LW --> CH & MN & LR & LDB
  API2 --> AN & RS & CD
  API -. docker.sock .-> HOST[(host Docker daemon: sandbox containers)]
```

How to read it: solid arrows are `depends_on` ordering, dotted arrows are runtime traffic. The default stack is only six services; everything Langfuse needs (five more containers) and the A2A multi-container mode are opt-in profiles.

Service notes (Verified from `docker-compose.yml`):

| Service | Role | Notable details |
|---|---|---|
| `postgres` | system of record: tasks, approvals, events, LangGraph checkpoints, memory (pgvector), demo telemetry schema | credentials `aeos/aeos` hard-coded; port 5432 published to the host; `pg_isready` healthcheck; volume `pgdata` |
| `redis` | provisioned, published on 6380 | `Settings.redis_url` exists but no code in `src/` references Redis (Verified by grep): unused infrastructure |
| `otel-collector` | OTLP receiver, fans traces to a debug exporter and Langfuse | distroless image so no shell healthcheck; liveness via `health_check` extension on :13133 (decision D-005 in `docs/DECISIONS.md`, as cited in a compose comment); requires `LANGFUSE_BASIC_AUTH` or compose refuses to start (`${VAR:?msg}`) |
| `aeos-seed` | one-shot: builds the demo git repo only if absent, then seeds demo telemetry data in Postgres | `command` uses `test -d ... || build_repo`, with a comment "never destroy an existing remote (it may hold branches of paused tasks)"; also creates the read-only DB role via seed DDL (`demo_target/seed.py`) |
| `api` | `aeos serve --host 0.0.0.0 --port 8000` with embedded worker | `restart: unless-stopped`; env anchored from `x-aeos-env`; mounts `demo_build` volume, `/var/run/docker.sock`, `./evals/results` read-only; host port 8800; healthcheck calls `/healthz` with Python `urllib`, `start_period: 20s`, 30 retries |
| `web` | Next.js dashboard | build arg `NEXT_PUBLIC_AEOS_API` (default `http://localhost:8800`); waits for `api` healthy |
| `langfuse-*` (profile `observability`) | Langfuse v4 stack: web, worker, ClickHouse 25.12, MinIO (chainguard), Redis 7, Postgres 17 | project, org, user and API keys are bootstrapped via `LANGFUSE_INIT_*` env from `.env`; all secrets use `${VAR:?}` required syntax |
| `analytics`, `research`, `coding`, `api-a2a` (profile `a2a`) | specialists as separate A2A services, the API delegating over the network | same image, different commands (`aeos agent <owner>`); `coding` and `api-a2a` also mount the Docker socket; `api-a2a` sets `AEOS_A2A_MODE=a2a` and the endpoint JSON |

Two topologies to name in an interview: "fast mode" (default; specialists in-process inside `api`) and "full topology" (`--profile a2a`; specialists are separate containers speaking A2A). See [16-a2a-and-github-integration](16-a2a-and-github-integration.md).

Compose-level observations (Verified unless noted):
- The `api` service does not depend on `otel-collector`; telemetry is best-effort. In the default stack Langfuse is not running, so the collector's Langfuse exporter has nowhere to send (Inferred: it retries then drops; `debug` exporter still logs).
- Pinned image tags: `pgvector/pgvector:pg16`, `otel/opentelemetry-collector-contrib:0.104.0`, Langfuse `:4`, `clickhouse-server:25.12`, `redis:7`, `postgres:17`. The chainguard MinIO image has no tag (floating `latest`) (Verified).
- `AEOS_LLM_PROVIDER` for `api` is `${AEOS_DASHBOARD_LLM:-gemini}`; the shared anchor default is `fake`. So the default dashboard uses live Gemini, and a free scripted demo needs `AEOS_DASHBOARD_LLM=fake` plus `AEOS_DASHBOARD_REPLAY=<scenario>` (what `make demo` sets with `wrong_fix`).
- Real GitHub publishing is on by default (`AEOS_GITHUB_PUBLISH` defaults to `true`) but only effective when repo and token are set; the compose comments say PRs for the seeded demo repo always stay local.

## 3. Configuration and secrets

- All settings are one `pydantic-settings` class (`core/config.py::Settings`): env prefix `AEOS_`, reads `.env`, `extra="ignore"`, `env_ignore_empty=True` (so an empty `${VAR:-}` from compose falls back to the default instead of becoming an empty string). Secrets are `SecretStr` (not shown in `repr`). Validators check the schema name and that database URLs start with `postgresql://`. `get_settings()` is `lru_cache`d (tests call `cache_clear`).
- `.env` is git-ignored; `.env.example` lists keys only. `scripts/init_env.py` copies `.env.example` to `.env` filling generated secrets (`secrets.token_hex`/`token_urlsafe`; Langfuse keys, salt, encryption key, Redis/MinIO passwords, A2A key; computes `LANGFUSE_BASIC_AUTH` as base64 of `public:secret`), refuses to overwrite, and `chmod 600`s the file. You add the Gemini key and GitHub token yourself.
- `scripts/check_no_secrets.py` (run by `make secrets`, `make check` and CI) walks the repo, skips `.env`, lockfiles, `node_modules` etc., and fails if (a) any value of a secret-named variable in your local `.env` appears in a file, or (b) a key-shaped string matches (Google, GitHub, AWS, private key block, OpenAI-style, Langfuse). It prints `file:line` and the variable *name*, never the value (Verified). Tested by `tests/test_no_secrets.py`.
- Hard-coded credentials that are *not* checked by that scanner because they are not key-shaped: `aeos/aeos`, `aeos_readonly/aeos_readonly`, ClickHouse `clickhouse/clickhouse`, Langfuse DB `postgres/postgres`, MinIO user `minio` (Verified in compose). Fine for local dev; flagged in section 8.

## 4. Startup sequence (default stack)

1. `postgres` starts; `pg_isready` healthcheck turns healthy (5 s interval, 10 retries).
2. `aeos-seed` (needs postgres healthy) builds the demo repo if missing and seeds demo data, then exits 0.
3. `api` starts (needs `aeos-seed` completed successfully): the FastAPI `lifespan` runs `ensure_schema()` for approvals, tasks and events (idempotent `CREATE ... IF NOT EXISTS`), opens the runtime and the Postgres checkpointer, then starts the embedded worker loop.
4. Compose healthcheck polls `/healthz` until it answers (up to ~20 s grace + 30 x 5 s).
5. `web` starts once `api` is healthy. `make up` uses `docker compose up -d --wait`, which blocks until services are healthy.
6. `otel-collector` and `redis` start independently (no dependents).

`scripts/check_infra.py` (`make infra-check`) parses `docker compose ps --format json` and exits 1 listing any service that is not `running` or whose `Health` is not healthy/empty; services with no healthcheck (the collector) count as fine when running. Pure function `unhealthy()` is unit-tested in `tests/test_check_infra.py` (3 tests).

Health checks only prove liveness: `/healthz` returns `{"status":"ok"}` without touching Postgres or the worker (Verified in `app.py`). There is no readiness endpoint that checks the DB, and a dead embedded worker task would not fail the healthcheck (Potential failure scenario, Inferred: the `run_forever` task is a background asyncio task whose crash is not surfaced by `/healthz`).

## 5. Tooling: uv, ruff, mypy, pytest

From `pyproject.toml` (Verified):
- Python `>=3.12`; build backend hatchling; console script `aeos = aeos.cli:main`.
- **uv** manages the venv and `uv.lock` (`uv sync`, `uv sync --frozen` in CI/Docker, `uv run ...`). Reasons (Inferred): fast, lockfile reproducibility, one tool for env + scripts.
- **ruff**: line length 100, rules `E, F, I, UP, B, ASYNC` (including async-misuse checks, relevant for an asyncio-heavy codebase); per-file `E501` ignores for prompt/SQL/fixture text. `make lint` = `ruff check .` and `ruff format --check .`.
- **mypy** with the pydantic plugin; **strict** only for `aeos.core.*` and `aeos.llm.*`; the rest uses default strictness; `docker` stubs ignored. `make typecheck` = `mypy src` (tests and web not type-checked by mypy; web has `tsc --noEmit` manually).
- **pytest**: `asyncio_mode=auto`; default `addopts = "-m 'not live and not chaos' -ra"` so tests that call a real LLM or kill -9 workers are opt-in; markers `live`, `docker`, `chaos`.
- `make check` = lint + typecheck + secrets + test. `make demo` generates `.env` if missing, then brings the stack up with the fake LLM replaying `wrong_fix`, with GitHub vars blanked so no real PR can occur.

## 6. CI (`.github/workflows/ci.yml`)

Triggers on push to `main` and on pull requests. One job, `ubuntu-latest`: checkout, `astral-sh/setup-uv@v5` with cache, `uv sync --frozen`, then `python scripts/check_no_secrets.py`, `make lint`, `make typecheck`, `make test`.

What it does not do (Verified by absence in the file):
- **No Postgres/service containers.** Many tests depend on the `seeded_db` fixture in `tests/conftest.py`, which calls `pytest.skip("Postgres not available")` when it cannot connect. So in CI those tests are *skipped*, not run: a green CI run does not exercise the store/API/graph tests that need a database (Inferred from fixture + workflow; I did not see a CI run). `CONTRIBUTING.md` itself says the slow graph tests take ~25 minutes with real sandboxes, so they are plausibly not meant for CI.
- No web build, `tsc`, or lint step; no Docker image build or compose smoke test; no dependency/vulnerability scan; no coverage; no deploy/release job; no matrix (Python 3.12 only).
- `python scripts/check_no_secrets.py` runs with the runner's system Python, not `uv run`; it is stdlib-only so this works.
- The only commit touching `.github` is the initial one (`git log -- .github`), so the CI has not evolved with the project (Verified).

## 7. Operating it (developer workflow)

`CONTRIBUTING.md` (Verified): `uv sync`, `python3 scripts/init_env.py`, `docker compose up -d postgres redis`; run tests against a throwaway `aeos_test` database because some tests clear the task queue; if the dev `.env` has GitHub repo+token set `AEOS_TARGET_ENABLED=false` for tests (otherwise default `kind` becomes `change` and the scripted LLM runs out of answers); do not run two test processes on one database; rebuild the web image after dashboard changes. `scripts/aeos` runs the CLI inside the `api` container if it is up (so no local Python is needed, and `AEOS_ACTOR` is forwarded) else through `uv run`.

## 8. What is missing for production (Recommendations, each grounded in a Verified fact)

| Area | Verified current state | Production recommendation |
|---|---|---|
| Docker socket | `api`, `coding`, `api-a2a` mount `/var/run/docker.sock` (compose comment calls it a documented trade-off) | Root-equivalent on the host. Use a rootless/remote Docker daemon, sysbox/gVisor, or a dedicated sandbox host/service (see [12-sandbox-and-execution-safety](12-sandbox-and-execution-safety.md)) |
| Container user | no `USER` in Dockerfile | run as non-root |
| Secrets | `.env` file, plain env vars, hard-coded DB creds, published DB port 5432 | secret manager, unique DB creds, do not publish DB/Redis/ClickHouse ports |
| TLS and auth | HTTP only; login optional and off by default; `AUTH_COOKIE_SECURE=false` default | reverse proxy with TLS, mandatory auth, secure cookies ([18-api-and-client](18-api-and-client.md)) |
| Scaling | single `api` replica with embedded worker; per-process sessions | separate `aeos worker` replicas (supported by `--no-worker`), shared auth secret, managed Postgres |
| Persistence/backup | named volumes only, no backup/restore, no migrations tool (idempotent `CREATE IF NOT EXISTS` DDL in each store) | managed Postgres + backups; Alembic-style migrations for schema changes |
| Health | liveness only; no DB readiness, no worker liveness | `/readyz` that checks DB and worker heartbeat |
| Observability | collector exports metrics to `debug` only; Langfuse optional and bound to local volumes | metrics backend + alerts, trace retention ([20-observability-and-telemetry](20-observability-and-telemetry.md)) |
| CI | see section 6 | Postgres service container, web `tsc`/build, image build + scan, release pipeline |
| Image hygiene | floating MinIO tag, full `node_modules` in web image | pin all tags/digests; Next `standalone` output |
| Resource limits | none in compose (no `mem_limit`, CPU, or sandbox quotas visible at this layer) | set limits; sandbox limits are covered in file 12 |
| Unused infra | Redis provisioned, unused | remove or use (e.g. event push, shared lockout) |
| Logging | default ERROR level in `serve`/`worker`; no log shipping | honour `AEOS_LOG_LEVEL`, ship JSON logs |

---

## Interview questions

## Q-DEP-1. Walk me through what happens when I run `docker compose up -d`.
**Difficulty:** Beginner  **Topic:** Startup sequence
**Short answer:** Postgres starts and becomes healthy, a one-shot `aeos-seed` container builds the demo repo and seeds data, then the `api` container starts, creates its tables, opens the checkpointer, starts an embedded worker, and passes its `/healthz` check, after which the `web` dashboard starts. The OTel collector and Redis start independently. Langfuse and the A2A services do not start unless their profiles are enabled.
**Detailed explanation:** Ordering is expressed with `depends_on` conditions: `service_healthy` (needs a healthcheck) and `service_completed_successfully` (for one-shot jobs).
**How this project uses it:** `docker-compose.yml`: `aeos-seed -> postgres healthy`, `api -> aeos-seed completed`, `web -> api healthy`. The API `lifespan` (`api/app.py`) runs `ensure_schema` for approvals, tasks and events.
**Example/walkthrough:** `make up` = `docker compose up -d --wait`; a missing `.env` var such as `LANGFUSE_SALT` aborts early because of `${VAR:?set X in .env}`.
**Follow-ups:**
- *Why a seed service instead of an init script in Postgres?* It reuses the app image and Python seeding code (`demo_target.seed`) and can guard "build repo only if absent".
- *What if the seed fails?* `api` never starts (`completed_successfully` is not met).
- *Why does the dashboard wait for the API?* So the first page load does not show "API offline".
**Trade-offs and alternatives:** Kubernetes init containers/jobs; Compose is enough for a single-host demo.
**Common interview mistake:** Assuming `depends_on` alone waits for readiness; it needs a healthcheck condition.
**Related:** [03-system-architecture](03-system-architecture.md), [18-api-and-client](18-api-and-client.md)

## Q-DEP-2. Why one Docker image for the API, workers and A2A services, and how is it built efficiently?
**Difficulty:** Intermediate  **Topic:** Dockerfile
**Short answer:** Every service is the same Python package with different `aeos` subcommands, so one image guarantees identical code. The Dockerfile installs locked dependencies in a cached layer before copying source, so code edits do not trigger a dependency reinstall.
**Detailed explanation:** `uv sync --frozen --no-dev --no-install-project` resolves only from `uv.lock`. Then `src/` is copied and `uv sync --frozen --no-dev` installs the project itself. `UV_COMPILE_BYTECODE` speeds startup.
**How this project uses it:** `Dockerfile`; compose services `api`, `analytics`, `research`, `coding`, `api-a2a` all use `image: aeos-app:dev` with different `command:`.
**Example/walkthrough:** Change `agents/loop.py`; rebuild reuses the dependency layer, only the source and install layers rerun.
**Follow-ups:**
- *Weaknesses?* Runs as root, no `HEALTHCHECK`, `git` included, the single image carries all dependencies for every role, and `demo_target`/`policy` are baked in.
- *Why `--frozen`?* Reproducible builds; mismatch between `pyproject` and lock fails loudly.
**Trade-offs and alternatives:** Separate slim images per role reduce attack surface; one image reduces drift.
**Common interview mistake:** Saying it is a multi-stage build; it is single-stage for Python.
**Related:** [02-repository-tour](02-repository-tour.md)

## Q-DEP-3. The API container mounts the Docker socket. Is that safe?
**Difficulty:** Advanced  **Topic:** Security trade-off
**Short answer:** It is a documented convenience, not a safe production pattern. Access to `/var/run/docker.sock` is effectively root on the host: anything that can talk to the API container's Docker client can start privileged containers. It exists so the app can create per-task sandbox containers. The sandbox restrictions limit what *agent code* can do, but not what the host-side manager can do.
**Detailed explanation:** The separation is: agents never execute on the host; the sandbox manager (running in the `api`/`coding` container) creates containers via the Docker API. A compromise of that manager process (rather than of the agent code inside a sandbox) would expose the host.
**How this project uses it:** `volumes: - /var/run/docker.sock:/var/run/docker.sock # creates sandbox containers` on `api`, `coding`, `api-a2a`; compose comments reference a trade-off in `docs/ARCHITECTURE.md`. Sandbox hardening is in [12-sandbox-and-execution-safety](12-sandbox-and-execution-safety.md).
**Example/walkthrough:** If the API had a code-execution bug, an attacker could `docker run -v /:/host ...` through the socket (Potential failure scenario, general Docker property).
**Follow-ups:**
- *How would you fix it?* Rootless Docker/Podman, a socket proxy that allows only create/start/rm on labeled containers, a separate sandbox runner service, or gVisor/Firecracker (Recommendation).
- *Does the `a2a` topology reduce exposure?* Only the `coding` service and `api-a2a` mount it, but `api-a2a` also does, so exposure is not reduced (Verified in compose).
**Trade-offs and alternatives:** Docker-in-Docker (privileged, also risky) vs socket mount vs remote daemon over TLS.
**Common interview mistake:** Claiming the sandbox makes the socket mount safe.
**Related:** [12-sandbox-and-execution-safety](12-sandbox-and-execution-safety.md), [14-security-and-prompt-injection](14-security-and-prompt-injection.md)

## Q-DEP-4. How are secrets handled, and how do you prove nothing sensitive is committed?
**Difficulty:** Intermediate  **Topic:** Secrets management
**Short answer:** Secrets live in a git-ignored `.env`, are loaded as `SecretStr`, are generated for local use by `init_env.py`, and are registered for redaction in traces. A scanner (`check_no_secrets.py`) fails the build if any value from your local `.env` or a key-shaped string appears in tracked files, printing only names and locations.
**Detailed explanation:** Two detection modes: exact-value matching (catches arbitrary-shaped secrets, but only on a machine that has the `.env`) and pattern matching (works in CI without a `.env`). In CI only the pattern mode can fire, because `.env` does not exist there.
**How this project uses it:** `scripts/check_no_secrets.py`, `scripts/init_env.py` (writes mode 0600), `Settings` `SecretStr`, `telemetry/setup.py::init_telemetry`, Makefile `secrets` target, CI step.
**Example/walkthrough:** Paste your Gemini key into a doc: local scan reports `docs/X.md:12: value of AEOS_GEMINI_API_KEY from .env`; the value is not printed.
**Follow-ups:**
- *Limitations?* Hard-coded dev passwords (`aeos/aeos`) are not key-shaped; the scanner skips `uv.lock`/`package-lock.json`; it is a point-in-time scan, not git-history scanning.
- *Why `.` separators in password hashes?* `$` is mangled by compose `.env` interpolation (comment in `api/auth.py`).
**Trade-offs and alternatives:** gitleaks/trufflehog with history scanning; a secret manager injected at runtime.
**Common interview mistake:** Saying secrets are "encrypted" - they are plain env vars.
**Related:** [20-observability-and-telemetry](20-observability-and-telemetry.md), [14-security-and-prompt-injection](14-security-and-prompt-injection.md)

## Q-DEP-5. What does your CI actually guarantee?
**Difficulty:** Advanced  **Topic:** CI honesty
**Short answer:** It guarantees formatting and lint (ruff), type checks (mypy, strict only in `core` and `llm`), the secret scan, and whichever pytest tests can run without Postgres or Docker. Because the workflow has no database service and the shared `seeded_db` fixture skips when Postgres is unreachable, DB-dependent tests are skipped in CI, so a green build says less than it appears to.
**Detailed explanation:** Skipping rather than failing is deliberate for local convenience, but in CI it hides coverage loss. The slow sandbox tests (~25 min per `CONTRIBUTING.md`) are plausibly intentionally local.
**How this project uses it:** `.github/workflows/ci.yml` (single job); `tests/conftest.py::seeded_db`; pytest markers `live`, `docker`, `chaos` excluded by default `addopts`.
**Example/walkthrough:** `tests/api/test_task_control.py` requires `seeded_db`; on a runner with no Postgres it is reported as skipped under `-ra`.
**Follow-ups:**
- *How to fix?* Add a `services: postgres: image: pgvector/pgvector:pg16` block with a health check and set `AEOS_DATABASE_URL`; fail on unexpected skips (e.g. `-p no:cacheprovider` plus a CI env that turns the skip into an error).
- *What else is absent?* Web typecheck/build, image build, vulnerability scan.
**Trade-offs and alternatives:** testcontainers; nightly job for slow tests.
**Common interview mistake:** "Everything is tested in CI". Say what is skipped.
**Related:** [22-testing-strategy](22-testing-strategy.md)

## Q-DEP-6. What do the health checks prove, and what do they not?
**Difficulty:** Intermediate  **Topic:** Liveness vs readiness
**Short answer:** `/healthz` returns a constant OK, so it proves the web server process is up. It does not check Postgres, the Docker daemon, or that the embedded worker loop is alive. Postgres uses `pg_isready`; the OTel collector uses an HTTP health extension port but, being distroless, has no in-container check; `check_infra.py` aggregates compose states.
**Detailed explanation:** Liveness answers "restart me?", readiness answers "send me traffic?". Mixed uses cause either needless restarts or black-holed traffic.
**How this project uses it:** `api` healthcheck: Python `urllib` to `/healthz`, 5 s interval, 30 retries, 20 s start period; `web` waits on it; `scripts/check_infra.py::unhealthy`.
**Example/walkthrough:** Worker task crashes with an unhandled exception: `/healthz` still OK, queued tasks never run until a restart (Potential failure scenario, Inferred; the lease/queue design would let a second worker take over if one existed).
**Follow-ups:**
- *Fix?* `/readyz` that does `SELECT 1` and checks the worker task is not done; use it for Compose and Kubernetes.
- *Why Python for the check?* The slim image has no curl.
**Trade-offs and alternatives:** Docker `HEALTHCHECK` in the image; external uptime probes.
**Common interview mistake:** Treating `/healthz` as a dependency check.
**Related:** [10-error-handling-and-reliability](10-error-handling-and-reliability.md)

## Q-DEP-7. Explain compose profiles and the two topologies.
**Difficulty:** Intermediate  **Topic:** Deployment modes
**Short answer:** The default stack is six services with specialists running in-process inside `api` (fast mode). `--profile a2a` additionally starts `analytics`, `research` and `coding` as separate containers and an `api-a2a` that delegates to them over the A2A protocol. `--profile observability` adds the Langfuse stack. Profiles keep optional heavy services out of a normal `up`.
**Detailed explanation:** Same image everywhere; only commands and env differ (`AEOS_A2A_MODE=a2a`, `AEOS_A2A_ENDPOINTS` JSON). `api-a2a` is on host port 8801 so both APIs can coexist.
**How this project uses it:** `docker-compose.yml` profiles `a2a` and `observability`; `Settings.a2a_mode` (`inprocess`/`a2a`).
**Example/walkthrough:** `docker compose --profile a2a up -d` then point the dashboard/CLI at `:8801`.
**Follow-ups:**
- *Why bother with separate containers?* Independent failure domains, per-service tool access and a realistic multi-agent network boundary; cost is latency and operational weight ([16-a2a-and-github-integration](16-a2a-and-github-integration.md)).
- *Do both APIs share Postgres?* Yes (same env anchor), so they share the task queue; running both with embedded workers means both can claim tasks (Potential failure scenario, Inferred; not tested).
**Trade-offs and alternatives:** Kubernetes with one Deployment per agent.
**Common interview mistake:** Thinking A2A mode is the default.
**Related:** [03-system-architecture](03-system-architecture.md)

## Q-DEP-8. What would you change first to make this production-grade?
**Difficulty:** Expert  **Topic:** Production readiness (Recommendation)
**Short answer:** In order: remove the Docker-socket trust problem (isolated sandbox runner), make authentication mandatory with TLS, use managed Postgres with backups and migrations, add readiness checks and a metrics/alerting backend, fix CI so database tests actually run, and run containers as non-root with pinned images and resource limits.
**Detailed explanation:** Ordering is by blast radius: the socket and open approval endpoint can lead to host compromise or unauthorised PRs; the rest is reliability and hygiene.
**How this project uses it:** table in section 8, each row tied to a Verified fact.
**Example/walkthrough:** n/a.
**Follow-ups:**
- *What already helps production?* Durable queue with leases and fencing, checkpointed resume, `restart: unless-stopped`, frozen lockfiles, secret scanning, a read-only DB role for analytics tools ([15-mcp-servers-and-tool-layer](15-mcp-servers-and-tool-layer.md)).
- *What would you not do?* Claim HA: there is one API replica and one Postgres.
**Trade-offs and alternatives:** Managed platform (ECS/Cloud Run/K8s) vs staying on Compose for a single-node tool.
**Common interview mistake:** Listing generic best practices not tied to this repo.
**Related:** [18-api-and-client](18-api-and-client.md), [20-observability-and-telemetry](20-observability-and-telemetry.md), [10-error-handling-and-reliability](10-error-handling-and-reliability.md)
