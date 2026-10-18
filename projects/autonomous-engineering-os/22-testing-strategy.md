# 22 - Testing strategy (tag: TEST)

## What this file covers

How AEOS is actually tested: the shape of the test pyramid as it exists on disk (not as the README describes it), the fakes and fixtures that make an LLM-driven system testable, what each major suite proves, how to run the tests without damaging your own dashboard data, the chaos (kill -9) suite, and, just as important for an interview, what is **not** tested. Evaluation of agent quality (as opposed to correctness of code) is a separate subject: see [23-evaluation-framework.md](23-evaluation-framework.md). Sandbox internals are in [12-sandbox-and-execution-safety.md](12-sandbox-and-execution-safety.md); durability in [07-state-durability-and-persistence.md](07-state-durability-and-persistence.md).

Verified facts (from reading `tests/`, `pyproject.toml`, `Makefile`, `.github/workflows/ci.yml`, `CONTRIBUTING.md`):

- 8,041 lines under `tests/`; 480 `def test_` functions; 24 `parametrize` uses expand that number. README says "About 530 tests (506 default, 27 chaos tests with real process kills)". I could not run the suite here (needs Postgres + Docker), so I treat 530 as the owner's reported figure and the 480 as what a static count shows.
- Default run: `addopts = "-m 'not live and not chaos' -ra"`, `asyncio_mode = "auto"`.
- Three markers: `live` (real LLM), `docker` (needs Docker), `chaos` (SIGKILL worker processes).

---

## 1. First principles: how do you test a system whose core is an LLM?

An LLM is non-deterministic, slow, and costs money. A test that calls it is flaky and expensive, and a green run tells you little. The standard answer has three parts:

1. **Put the model behind an interface** so a scripted fake can replace it (a "test double").
2. **Test everything around the model deterministically**: the loop that calls it, the policy that gates its tools, the verifier that checks its output. These are ordinary code with ordinary assertions.
3. **Measure the model's actual quality separately**, statistically, on a dataset (that is evals, not tests).

AEOS does exactly this. Tests answer "does the machinery behave correctly given *any* model output, including garbage?" Evals answer "how often does the real model produce good outcomes?"

---

## 2. The pyramid as it actually exists

Counts are `def test_` per directory (static `grep`, before parametrisation).

| Layer | Directories (test fn count) | Needs | Speed |
|---|---|---|---|
| Pure unit | `core` (41), `memory` gate/embedder, `agents` loop/judge/replan/supervisor (65 total in `agents`), `llm` (13), `mcp/test_sql_guard.py`, `tests/evals/test_evals_unit.py`, `client` (63) | nothing | milliseconds |
| Integration with Postgres | `store` (20), `api` (15), most of `memory` (42), `mcp` DB/GitHub/observability (66), `telemetry` (10) | Postgres (`seeded_db` fixture) | seconds |
| Integration with Docker | `sandbox` (37), `agents/test_verifier_gates.py`, `test_verifier_change.py` | Docker daemon | seconds each |
| Full graph end to end | `graph` (45): `test_e2e`, `test_hitl`, `test_replan_e2e`, `test_change_e2e`, `test_memory_e2e`, `test_trace_e2e` | Postgres + Docker + real MCP child processes | slow: CONTRIBUTING says about 25 minutes for `tests/graph` |
| Multi-process | `a2a` (11): real specialist processes over HTTP; `tests/evals/test_evals_smoke.py` | Postgres + Docker | slow |
| Chaos | `chaos` (3 functions, 27 cases) | Postgres + Docker + subprocess workers | about 13 minutes (PROGRESS.md) |
| Live | `llm/test_live_gemini.py` (2 tests) | Gemini API key | excluded by default |

```mermaid
flowchart TB
    L[live: 2 tests, real Gemini, excluded by default]
    C[chaos: 27 cases, kill -9 real workers]
    E[graph e2e + a2a + eval smoke: real MCP, real sandbox, scripted LLM]
    I[integration: Postgres, Docker sandbox, MCP servers]
    U[unit: policy, loop, gates, gate rules, graders, client rendering]
    U --> I --> E --> C --> L
```

How to read it: width would be proportional to number of tests; the bottom is wide and fast. Note the unusual feature: the *middle* layer (full graph with real tools and a scripted model) is unusually heavy, because AEOS deliberately avoids mocking MCP servers, Postgres or Docker (D-014 and D-017 both came from real failures, not mocks).

---

## 3. Fakes and fixtures (the test infrastructure)

| Component | File | What it gives you |
|---|---|---|
| `FakeLLM` | `src/aeos/llm/fake.py` | Plays a scripted list of `ScriptStep`s in order. Fails loudly: `ScriptExhaustedError` if more calls than script, `UnexpectedPromptError` if a step's `expect_in_prompt` does not match. Records calls (`llm.calls`) so tests can assert what the model was shown. |
| `RoutedFakeLLM` | same | One script per role (planner, research, coding, judge, memory...), chosen by system-prompt match, so *parallel* agents stay deterministic. `remaining` shows unconsumed turns. |
| Golden trajectories | `src/aeos/demo/trajectories.py` | `happy_path_llm`, `wrong_fix_then_fix_llm`, `coding_script_good_fix`, `replan_script`... Used by tests, by `aeos run --replay`, by `make demo`, and by the eval harness' fake mode. It lives in `src` not `tests`, so the product demo and the tests share one source of truth. |
| `turn()`, `echo_tool()`, `make_runtime()` | `tests/agents/helpers.py` | Build one scripted model turn; a trivial tool with configurable `risk` and `origin`; a `ToolRuntime` with a real `PolicyEngine` and in-memory `ListSink`. |
| `seeded_db` | `tests/conftest.py` (session scope) | Ensures the demo schema exists and 1,440 metric rows are loaded; **skips** (does not fail) if Postgres is unreachable (`connect_timeout=2`). |
| `remote_template` | `tests/conftest.py` | Builds the 20-commit demo repo once, then each test copies it to a fresh bare remote. Tests that write branches never share a remote. |
| `memory_off_by_default` | `tests/conftest.py` (autouse) | Sets `AEOS_MEMORY_ENABLED=false` and `AEOS_TELEMETRY_ENABLED=false` so tests never export spans or touch long-term memory unless they opt in. |
| `spans` | `tests/conftest.py` | In-memory OpenTelemetry exporter to assert on the span tree. |
| `mgr` / `sbx` | `tests/sandbox/conftest.py` | Session `SandboxManager` (skips if `docker.from_env().ping()` fails; `reap(keep=set())` at teardown) and a per-test sandbox. |
| MCP fixtures | `tests/mcp/conftest.py` | A real stdio `McpClient` to the database server; a factory `gh()` for a GitHub MCP client bound to a fresh remote (truncates `vcs.pull_requests`). |
| `tests/test_no_network.py` | | Monkeypatches `socket.connect` and `getaddrinfo` to raise, then proves `FakeLLM` makes zero network calls. |

### Why scripted fakes can still be dangerous (Observed limitation)

`FakeLLM` only proves the system behaves correctly for the scripted turns. A fake that returns a perfect fix cannot reveal that the real model would not find it. This is why the numbers in `evals/results` from live Gemini runs matter more than 100% pass in the scripted suite; see [23-evaluation-framework.md](23-evaluation-framework.md).

---

## 4. What each major suite verifies

### 4.1 Agent loop (`tests/agents/test_loop.py`, 191 lines)
Tool-then-finish; recovery from a model turn with no tool call; repeated malformed output stops with status `malformed`; an invalid `finish` is fed back (`"INVALID finish"`) then accepted; max-steps halts a runaway loop (50 scripted turns, `max_steps=5`); the token budget halts the loop; the step budget is shared across agents; the tool result reaches the model labelled `origin="untrusted"`. This is the guardrail-in-code evidence for loop limits.

### 4.2 Verifier gates (`tests/agents/test_verifier_gates.py`, real sandboxes on the demo repo)
- The real fix passes all gates (`patch_present`, `patch_applies`, `tests_pass`, `lint_ok`, `benchmark_not_regressed`, `diff_size_ok`, `paths_allowed`, `no_secrets_in_diff`).
- The red-herring fix (change `AUDIT_EVERY` 25 to 1000) passes tests and paths but **fails the benchmark gate** with a `perf_regression` issue.
- "Cheating" by editing `checkout/costs.py` (the cost model) is caught by `paths_allowed` (`forbidden_edit`).
- A broken test is reported by name; a syntax error fails lint and tests; a patch that does not apply is a clean failure; an empty patch fails without touching Docker.
- `test_a_baseline_that_is_not_faster_than_the_head_cannot_verify_anything`: its docstring says "Found by the live eval". A wrong "last good" commit let the benchmark gate pass trivially, so a `regression_reproduced` gate was added. This is a real example of eval findings feeding tests (see [24-technical-challenges-and-history.md](24-technical-challenges-and-history.md)).

### 4.3 Sandbox abuse (`tests/sandbox/test_abuse.py`, 144 lines)
Everything hostile runs inside the container. Tests: infinite loop killed with the sandbox surviving; fork bomb contained (`PidsLimit == 128`) with no stragglers or sandbox destroyed; background straggler (`start_new_session=True`) killed; memory hog OOM-killed; workspace-full gives `ENOSPC`; **network egress impossible** (DNS, HTTP, raw TCP all "blocked" and `/proc/net/dev` shows only `lo`); writes outside `/workspace` denied; host paths and Docker socket invisible; runs as uid 10001 with `CapEff == 0000000000000000` and `setuid(0)` denied; 10 MB output flood capped (`truncated`, <= 1 MB); a symlink to `/` resolves inside the container.

Honest scope: these prove the hardening flags do what they claim on this host. They are not a container-escape test (no kernel exploits) and a passing run says nothing about a kernel or runtime vulnerability.

### 4.4 SQL guard (`tests/mcp/test_sql_guard.py`)
8 allowed statements and 26 forbidden ones in parametrised tables: DROP/UPDATE/DELETE/INSERT/TRUNCATE/ALTER/CREATE/GRANT/COPY/SET ROLE, stacked statements, a comment before a write, a DELETE inside a CTE, `SELECT ... INTO`, `FOR UPDATE`, and function denylist hits (`pg_sleep`, `pg_read_file`, `lo_import`, `set_config`, `nextval`, `query_to_xml`), including a nested one. The second layer (read-only Postgres role) is tested separately in `tests/mcp/test_database.py`.

### 4.5 Risk policy (`tests/core/test_risk_policy.py`, 189 lines)
Decision matrix; large diffs need approval; untrusted-origin writes are gated; aliasing and case tricks cannot bypass deny; a renamed irreversible tool is still gated by its *declared* risk; approval grants do not transfer to different arguments or a different tool; **a grant can never override a deny**; unknown decisions or match keys fail loudly instead of silently never matching.

### 4.6 Human-in-the-loop (`tests/graph/test_hitl.py`, 474 lines)
The headline test, `test_run_pauses_process_dies_later_approval_resumes_without_repeating_work`: process 1 submits a task and pauses at `create_pull_request`; its app is torn down; a *second* runtime whose model has nothing scripted resumes on approval by `alice`; the test asserts the restarted model was never called, exactly one PR exists, and the audit trail is `approval_requested`, `approval_approved`, `pr_created` with the right actors. Others: rejection with feedback produces a revised plan; repeated rejection escalates; decisions are exactly-once; concurrent approvals create one PR; expired approvals return failure and escalate; tampering with approved arguments is blocked; no approver means the run stays paused.

### 4.7 Store layer (`tests/store/`)
`enqueue` idempotent; `claim` uses lease; **8 threads race and exactly one claims** (`test_only_one_of_many_concurrent_workers_claims_a_task`); oldest first; dead worker's lease expires and another takes over (per the test names; lease and fencing in `store/tasks.py`).

### 4.8 API auth (`tests/api/test_auth.py`)
Password hash roundtrip; session cookie rejects tampering and expiry; gate blocks everything except health/auth; a logged-in name cannot be spoofed with `X-Actor`; service token names itself; repeated failures lock the account even for the right password; 401 still carries CORS headers; without users configured, nothing changes.

### 4.9 Memory (`tests/memory/`, 42 tests; `tests/graph/test_memory_e2e.py`)
Write gate rules (secrets, instruction-like text such as "ignore all previous instructions", low confidence, duplicates, untrusted-only provenance), supersession, retrieval order, token budget; end to end: first run writes memory, second recalls it; **memory failure never fails the task**; poisoned curator proposals refused.

### 4.10 Terminal client (`tests/client/`, 63 tests)
Pure render functions (API JSON in, rich renderable out), sanitizer for terminal escape codes (`test_sanitize.py`), command handling, and `test_e2e_terminal.py` (real server, SSE, inline approval).

### 4.11 Demo target (`tests/demo_target/`)
The test fixture itself is tested: SHAs reproducible across rebuilds; the regression raises p95 at least 2.5x; the real fix restores the baseline; the red-herring-only fix does **not** pass the p95 gate; ground-truth numbers match a real benchmark run. If these fail, every eval result would be meaningless, so they are the foundation of the eval's validity.

### 4.12 Telemetry and A2A
`tests/telemetry/test_tracing.py` (secrets never reach a span, trace context crosses the MCP process boundary, `approval.wait` has explicit wall-clock bounds). `tests/a2a/test_a2a.py`: Agent Cards, a generic A2A client drives a specialist, bearer-token enforcement, full task through three separate processes, killing the only research service escalates without hanging, a second instance takes over, silent service declared dead after the stall timeout, deadline cancels remote work, trace context and budget cross the boundary.

### 4.13 Repo hygiene
`tests/test_no_secrets.py` runs `scripts/check_no_secrets.py` over the repository and asserts `scan() == []`. `tests/test_check_infra.py` unit-tests the compose health parser.

---

## 5. The chaos suite (`tests/chaos/test_chaos.py`)

Concept: *chaos testing* kills components at unpredictable (or exhaustively chosen) moments and checks that invariants still hold. AEOS's invariant: **a task killed anywhere still completes, with no duplicated side effect.**

Mechanism:
1. `src/aeos/failure/crash.py::maybe_crash(point)` sends `SIGKILL` to its own process if `AEOS_CRASH_AT` names the point. It is a no-op otherwise. `SIGKILL` gives no cleanup, which is what a real `kill -9` or OOM kill does.
2. The test enqueues a task and spawns worker A with `AEOS_CRASH_AT=<point>` (`python -m aeos.cli worker --replay happy --yes --lease 3 ...`). It asserts the exit code is `-SIGKILL`.
3. It spawns worker B. After A's lease (3 s in tests) expires, B claims the task and resumes from the last Postgres checkpoint.
4. `assert_exactly_once`: status `completed`; `attempts >= 2`; exactly one row in `vcs.pull_requests`; exactly one branch `aeos/<task>-v1`; the branch has exactly one more commit than `main`; one approval request; audit shows one each of `approval_requested`, `approval_approved`, `pr_created`; no container labelled with the task id remains (the crashed worker's sandbox is reaped).

The matrix: `POINTS` = 8 graph nodes x before/after (16) + 7 tool boundaries (`list_deployments:after`, `sandbox_write_file:after`, `sandbox_compare_to_baseline:before`, `create_branch:after`, `commit_patch:after`, `create_pull_request:before|after`) = 23 parametrised cases; plus 3 random-kill seeds (three SIGKILLs at random 3 to 14 s intervals, then a final worker); plus 1 crash-while-awaiting-approval test (worker killed while paused, restarted, human approves, completes). 23 + 3 + 1 = 27, which matches PROGRESS.md "27/27 pass" and README "27 chaos tests".

```mermaid
sequenceDiagram
    participant T as Test
    participant A as Worker A (AEOS_CRASH_AT=point)
    participant PG as Postgres
    participant B as Worker B
    T->>PG: enqueue task
    A->>PG: claim (lease 3s)
    A->>A: runs graph, checkpoint after each node
    A--xA: SIGKILL at named point
    Note over PG: lease expires
    B->>PG: claim (attempts = 2)
    B->>PG: resume from last checkpoint
    B->>PG: finish, one PR
    T->>PG: assert exactly-once invariants
```

How to read it: the only communication between A and B is the database; that is the whole durability argument. What the matrix does **not** cover: Postgres itself dying, Docker daemon dying, partial writes of a non-Postgres external service (the real GitHub mirror), clock skew between workers.

Bugs the chaos suite found (PROGRESS.md Phase 7, all fixed): `create_branch` replay after our own commit reported a conflict; `pr_created` audit entry written twice if the worker died between PR creation and checkpoint; a crashed worker's sandbox lingered until the next sweep.

---

## 6. How to run tests safely

**The hazard (documented in README and CONTRIBUTING):** the suite shares the development database by default, and some tests clear the task queue (`DELETE FROM aeos.tasks` in `tests/store/test_tasks.py` and `tests/api/test_task_control.py`; the chaos fixture deletes `queued`/`running` tasks). Running `make test` while using the dashboard can wipe your queue.

Safe procedure from `CONTRIBUTING.md`:

```text
docker compose exec -T postgres psql -U aeos -d aeos -c "CREATE DATABASE aeos_test"
export AEOS_DATABASE_URL=postgresql://aeos:aeos@localhost:5432/aeos_test
export AEOS_READONLY_DATABASE_URL=postgresql://aeos_readonly:aeos_readonly@localhost:5432/aeos_test
uv run pytest tests/api tests/store tests/core tests/agents     # fast
uv run pytest tests/graph                                       # slow, real sandboxes
uv run pytest -m chaos tests/chaos                              # explicit opt-in
uv run pytest -m live                                           # costs money
```

Extra rules from CONTRIBUTING: if `.env` has `AEOS_GITHUB_REPO` and `AEOS_GITHUB_TOKEN`, set `AEOS_TARGET_ENABLED=false` or an unkinded task defaults to a "change request" on the real repo and graph tests fail with "FakeLLM script exhausted". Do not run two test processes against one database.

Is it safe for GitHub? The e2e tests use a local bare remote (`AEOS_GIT_REMOTE_PATH`); the real-GitHub path (`mcp_servers/github/remote.py`) is tested against a fake HTTP server in `tests/mcp/test_github_remote.py` (class with `html_url: https://github.com/owner/repo/pull/...` is test data, not network).

---

## 7. What is not tested (honest list)

| Gap | Evidence | Severity |
|---|---|---|
| **No frontend tests.** `web/package.json` has no `test` script and no jest/vitest; CONTRIBUTING's only check is `npx tsc --noEmit`. | grep of `web/package.json` | Medium: type-checking only |
| **Default CI probably skips most DB-backed tests.** `ci.yml` runs `make test` with no `services:` Postgres. `seeded_db` skips when Postgres is unreachable, and `tests/store`, `tests/api`, `tests/mcp` and `tests/graph` depend on it. | `.github/workflows/ci.yml`; `tests/conftest.py` | Inferred: PROGRESS.md itself says "CI unverified remotely" |
| **The `docker` marker does not auto-skip.** `pyproject.toml` says "skipped when unavailable", but there is no `pytest_collection_modifyitems`; skipping happens only in `tests/sandbox/conftest.py` and `test_verifier_gates.py`. Graph tests would error, not skip, without Docker. | grep of `tests/` | Low: confusing, not unsafe |
| **Live model behaviour is only 2 tests** (`tests/llm/test_live_gemini.py`: tool-call roundtrip, structured output). The Gemini provider's retry-on-429/5xx logic (`llm/gemini.py`) has no test I could find; `test_gemini_conversion.py` covers message conversion, parallel tool results, response parsing and the blocked/empty-candidates case only. | file read | Medium |
| **No direct test for `telemetry/metrics.py`**, `scripts/init_env.py`, `scripts/mcp_probe.py`. | grep of `tests/` | Low |
| **Prompt-injection: targeted, not an adversarial suite.** `docs/PROMPT_INJECTION.md` says "There is no large adversarial test set". | doc | Medium (ROADMAP Phase 13 asked for one) |
| **Fault injection beyond crashes** (MCP timeout, LLM rate limit, DB down, malformed tool output as an injectable fault) has no injector and no table. Some of these are covered individually (A2A stall, flaky specialist retry in `test_replan_e2e.py`, `McpClient` timeouts). | `src/aeos/failure/` has only `crash.py` | Medium vs ROADMAP Phase 12 |
| **Single-host scale**: no load or concurrency tests beyond 8-thread claim races. | | Low for a prototype |
| **No coverage measurement** configured (no `pytest-cov` in `pyproject.toml`). | `pyproject.toml` | Low |

---

## Q-TEST-1. How do you test a system built around a non-deterministic LLM?
**Difficulty:** Intermediate  **Topic:** Test doubles, determinism
**Short answer:** "I separate correctness from quality. Correctness: the model sits behind an interface, and `FakeLLM` plays scripted turns, failing loudly if the script is exhausted or the prompt is unexpected. Everything around the model, the loop, risk policy, verifier, store, is ordinary deterministic code with ordinary tests. Quality, how good the real model is, is measured separately with evals on a 30-task dataset with multiple graders."
**Detailed explanation:** The risk with fakes is that they only prove the machinery. So AEOS runs real MCP servers, real Postgres and real Docker sandboxes under the scripted model, so the *integration* is real even when the model's words are scripted.
**How this project uses it:** `llm/fake.py` (`FakeLLM`, `RoutedFakeLLM`), `demo/trajectories.py`, `tests/agents/helpers.py`. A property worth citing: `test_no_network.py` proves `FakeLLM` opens no socket.
**Example/walkthrough:** `test_max_steps_halts_a_runaway_loop` scripts 50 tool calls, limits to 5 steps, asserts 5 recorded tool calls and status `max_steps`. No real model could be relied on to misbehave on cue.
**Follow-ups:**
- *What does a passing scripted suite not tell you?* Whether the real model finds the bug; the Oct 3 live run solved 15/30 while the scripted suite was 100% (`evals/results/20261003T024031Z_live-gemini-full.md` vs `...smoke-healthy.md`).
- *Why RoutedFakeLLM?* Parallel specialists would consume a single script in nondeterministic order; routing by system prompt gives each role its own queue.
**Trade-offs and alternatives:** Record/replay of real model outputs (VCR-style) gives realism but goes stale when prompts change; AEOS hand-writes golden trajectories instead.
**Common interview mistake:** Saying "we mock the LLM so we test it". Mocking proves the plumbing only.
**Related:** [06-llm-integration.md](06-llm-integration.md), [23-evaluation-framework.md](23-evaluation-framework.md)

## Q-TEST-2. Explain your chaos tests. What exactly do they prove?
**Difficulty:** Advanced  **Topic:** Durability testing
**Short answer:** "They SIGKILL a real worker subprocess at 23 named points, every graph node before and after, plus tool boundaries, plus random kills, plus a kill while awaiting human approval. A second worker takes over after the lease expires and must finish with exactly one PR, one branch, one extra commit, one approval, one audit entry per effect, and no leftover container. It proves at-least-once execution with exactly-once effects for those points on a single Postgres."
**Detailed explanation:** The crash hook is `maybe_crash`, driven by `AEOS_CRASH_AT`. It deliberately does no cleanup. Exactly-once is not magic: it comes from idempotency keys (branch, commit trailer, unique PR key, audit `dedupe_key`) and from re-running the interrupted node (D-032).
**How this project uses it:** `tests/chaos/test_chaos.py::test_kill_9_at_every_point_still_completes_exactly_once`, `assert_exactly_once`.
**Example/walkthrough:** Kill at `tool:create_pull_request:after`: PR exists, checkpoint not yet written. Worker B re-runs `publish_pr`; the PR idempotency key replays the existing PR instead of making a second one; audit `pr_created` is deduped (this was exactly bug 2 in PROGRESS.md).
**Follow-ups:**
- *What did it find?* Three bugs listed above, all fixed. (Verified in PROGRESS.md.)
- *What is not covered?* Postgres or Docker dying; the real GitHub mirror; clock skew. (Recommendation: add a test with a stalled DB connection.)
- *Why a 3-second lease in tests?* So takeover is quick; `Worker.__init__` defaults `lease_s` to 30.0 seconds.
**Trade-offs and alternatives:** Temporal gives durable execution as a service (D-033 rejects it for operational cost at this scale); Jepsen-style fault injection is more thorough but heavier.
**Common interview mistake:** Claiming "exactly-once processing". It is at-least-once with idempotent effects.
**Related:** [07-state-durability-and-persistence.md](07-state-durability-and-persistence.md), [10-error-handling-and-reliability.md](10-error-handling-and-reliability.md)

## Q-TEST-3. Why not mock Postgres, Docker and MCP in the end-to-end tests?
**Difficulty:** Intermediate  **Topic:** Integration vs mock
**Short answer:** "Because the bugs were in the seams. DECISIONS D-014 (anyio cancel scopes, found via a real failure) and D-017 (docker `put_archive` refuses read-only rootfs) are the kind of thing a mock would hide. The cost is slow tests: tests/graph takes about 25 minutes per CONTRIBUTING."
**Detailed explanation:** Mocks encode your assumptions about the dependency; real dependencies correct them.
**How this project uses it:** `tests/graph/test_e2e.py` opens a real runtime (`open_runtime`) with real child-process MCP servers and a real sandbox, scripted LLM only.
**Example/walkthrough:** `test_happy_path_completes_and_everything_is_recorded` asserts every scripted turn was consumed (`llm.remaining` all zero except memory) and no sandbox is left (`leftover == []`).
**Follow-ups:**
- *How do you keep the fast loop fast?* `tests/api tests/store tests/core tests/agents` per CONTRIBUTING; graph tests run on demand.
- *What breaks CI?* See section 7: CI has no Postgres service.
**Trade-offs and alternatives:** Testcontainers per test class; contract tests; mocks for the fast layer and a nightly real layer.
**Common interview mistake:** Treating slow tests as a smell by default.
**Related:** Q-TEST-5

## Q-TEST-4. How do you verify the sandbox really is isolated?
**Difficulty:** Advanced  **Topic:** Security testing
**Short answer:** "By running hostile code inside it and asserting on observable facts: only the loopback interface exists, DNS and TCP fail, the root filesystem is unwritable, uid is 10001 with an empty capability set, pids are limited to 128, and a fork bomb or memory hog is contained. These prove the hardening settings work on this host; they are not a kernel-exploit test."
**Detailed explanation:** The tests are black-box from inside the container, and the hostile code never runs on the host (module docstring: "Everything hostile runs INSIDE the container, never on the host").
**How this project uses it:** `tests/sandbox/test_abuse.py`; hardening in `sandbox/manager.py`; hardened-container attributes also asserted in `test_manager.py::test_container_is_hardened`.
**Example/walkthrough:** `test_network_egress_is_impossible` tries `gethostbyname`, `urlopen('http://1.1.1.1')`, `create_connection(('8.8.8.8', 53))`, expects `["blocked"]*3`, then reads `/proc/net/dev` and expects `["lo"]`.
**Follow-ups:**
- *What escapes would this miss?* A runtime or kernel vulnerability; a misconfigured Docker socket mount in the *coding service* (D-041 documents that compose mounts the Docker socket there).
- *Is the npm container tested?* Change-request tests exercise it (`test_verifier_change.py`, `test_change_e2e.py`).
**Trade-offs and alternatives:** gVisor/Firecracker give a stronger boundary at higher cost; AEOS uses plain Docker with dropped caps and no network.
**Common interview mistake:** "It's secure because we have tests." The tests show the configuration is applied, not that Docker is unbreakable.
**Related:** [12-sandbox-and-execution-safety.md](12-sandbox-and-execution-safety.md)

## Q-TEST-5. Where are the weak spots in your test strategy?
**Difficulty:** Expert  **Topic:** Self-critique
**Short answer:** "Three: the dashboard has only type-checking, no tests; the default CI workflow has no Postgres service so DB-backed tests would skip there, which means the strongest suites are run locally; and the prompt-injection tests are targeted rather than a red-team set. I also have only two live-model tests. I say these upfront because they are documented in the repo's own known-limits sections."
**Detailed explanation:** See section 7 for evidence. CI skip behaviour is inferred from `ci.yml` and `seeded_db`, and PROGRESS.md states CI was not verified remotely.
**How this project uses it:** `CONTRIBUTING.md` ("Keep known limits honest"), `docs/PROMPT_INJECTION.md` "What is not covered".
**Example/walkthrough:** To fix CI: add a `postgres` service with pgvector to the job, set `AEOS_DATABASE_URL`, and run `tests/api tests/store tests/core tests/agents` (Recommendation).
**Follow-ups:**
- *What would you add first?* Postgres in CI (cheap, big win), then a Playwright smoke test for the approval flow, then an adversarial set seeded into `demo_target`.
- *Coverage?* Not measured; I would add `pytest-cov` but would not chase a percentage.
**Trade-offs and alternatives:** More mocks make CI fast but weaken the guarantees the project is built on.
**Common interview mistake:** Hiding gaps. Interviewers probe them; being first to name them reads as maturity.
**Related:** [24-technical-challenges-and-history.md](24-technical-challenges-and-history.md)

## Q-TEST-6. What does the "approval resume" test prove, and how would you break it?
**Difficulty:** Advanced  **Topic:** Human-in-the-loop correctness
**Short answer:** "That a paused task survives its process dying and resumes from the checkpoint with no repeated model calls and exactly one PR. The second runtime's LLM has nothing scripted, so any model call would raise. To break it I would make `publish_pr` re-plan or rerun earlier nodes, or lose the stored decision between process 1 and 2."
**Detailed explanation:** It also checks the audit sequence and actors, and that the approval's evidence (gates, patch, benchmark `p95_change_pct <= 10`) is visible.
**How this project uses it:** `tests/graph/test_hitl.py::test_run_pauses_process_dies_later_approval_resumes_without_repeating_work`; the `silent_llm()` helper and `all(not llm.calls for llm in silent.routes.values())`.
**Example/walkthrough:** Compare with `test_tampering_with_the_approved_arguments_is_blocked`: approval is bound to a hash of the exact PR arguments (D-030).
**Follow-ups:**
- *Is that a real process death?* Process 1 is torn down by leaving the `running_app` context (same pytest process); the real `kill -9` version is in the chaos suite (`test_crash_while_awaiting_approval_then_restart_then_approve`).
**Trade-offs and alternatives:** In-process teardown is faster and deterministic; subprocess kill is more faithful.
**Common interview mistake:** Calling the in-process test a crash test.
**Related:** [13-verification-and-risk-policy.md](13-verification-and-risk-policy.md), Q-TEST-2
