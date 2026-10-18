# 24 - Technical challenges and history (tag: HIST)

## What this file covers

What the git history, `docs/PROGRESS.md`, `docs/DECISIONS.md`, the eval result files and `ROADMAP.md` actually show about how AEOS was built: a timeline, the specific problems that are documented in the repo, each labelled by how strong the evidence is, how to narrate the project honestly in an interview, and where the original ROADMAP and the implemented system differ. Nothing here is a personal anecdote: every item cites a commit, file or test. Related: [22-testing-strategy.md](22-testing-strategy.md), [23-evaluation-framework.md](23-evaluation-framework.md), [02-repository-tour.md](02-repository-tour.md).

Evidence labels used below:

- **Verified historical issue**: a commit message, test docstring, PROGRESS/DECISIONS entry or result file says it happened.
- **Observed limitation**: visible in the code or results today, no claim about history.
- **Potential scenario**: plausible failure not shown to have happened.
- **Suggested discussion**: a design question worth raising yourself.

---

## 1. Timeline (from `git log`, 8 commits, one author)

| Date (2026) | Commit | What it contains |
|---|---|---|
| 2 Oct | (no commit) | Plans in `docs/superpowers/plans/2026-10-02-*`; first eval result files `20261002T165442Z_dev-check`, `...165544Z_smoke-healthy` |
| 3 Oct | (no commit) | Recovery runs (healthy and `--break retry`) and the first full live Gemini run `20261003T024031Z_live-gemini-full` |
| 4 Oct | `d9cf6ea` "added-autonomous-codeforge-layer" | One large snapshot: the whole Python package (a2a, agents, api, graph, mcp_servers, memory, sandbox, store, telemetry...), tests, evals, `demo_target`, compose, docs, ROADMAP is *not* yet here |
| 11 Oct | `e002186` | Terminal client (`src/aeos/client/*`, 8 test files), README, ROADMAP.md (640 lines), INTERVIEW_PREP.md |
| 12 Oct | `d663736` | Dashboard redesign: nav, command palette, pipeline, DESIGN.md, screenshots |
| 14 Oct | `f14683e` | Optional login (scrypt, signed cookie, lockout, service token), task cancel/retry, `/evals` endpoints |
| 14 Oct | `126c990` | Verifier runs the repo's own tests for change requests (`agents/suite.py`) |
| 14 Oct | `2fab6fc` | `fix(evals)`: save results before comparing; adds the 12 Oct live run |
| 14 Oct | `65a08b4` | Web: trace waterfall, evals page, login UI, cancel, diff comments |
| 14 Oct | `2cd8ad1` | README, CONTRIBUTING, PROMPT_INJECTION.md, DECISIONS D-046 to D-050 |

Important reading of this table (Observed):

1. **The git history is not a development log.** One commit contains roughly everything from phases 0 to 10 and the evals. The real chronology of phases lives in `docs/PROGRESS.md`, the dated plan files and the timestamps in `evals/results/`. Say "the repo history was committed in snapshots" rather than implying incremental commits.
2. **The five 14 Oct commits share an identical timestamp** (`10:22:03 +0530`). That is consistent with being created in one batch (Inferred; the cause is not recorded). Do not present them as five separate working sessions.
3. **Result files predate the first commit.** The 3 Oct live run's version hashes (`code 61faa5ca5e9d`) describe code that is not a git commit; you cannot check out the code that produced the 50% result. The 12 Oct run (`code 02fe54dfda03`) likewise predates `2fab6fc` by two days (Observed from timestamps).
4. **Metadata quirk:** D-003 says "No git operations by the agent (user decision)... The user manages git/remotes themselves". That governs the agent and the earlier assistant workflow, and the repo was committed later; do not quote D-003 as describing the repo's history.

---

## 2. Documented challenges

### 2.1 Chaos suite found three exactly-once bugs
**Label:** Verified historical issue. **Evidence:** `docs/PROGRESS.md` Phase 7 ("What the chaos tests found (all fixed)"): (1) `create_branch` replay after our own commit moved the branch reported a conflict, so the idempotency key is now recorded and honoured; (2) the `pr_created` audit entry was written twice when the worker died between creating the PR and checkpointing, so audit entries carry `dedupe_key`; (3) a crashed worker's sandbox lingered, so the worker reaps a task's sandboxes when the task ends. **Test:** `tests/chaos/test_chaos.py` (27 cases). **Lesson:** exactly-once effects need idempotency at every boundary; the checkpoint does not make a side effect atomic with it.

### 2.2 anyio cancel scopes broke the MCP client
**Label:** Verified historical issue. **Evidence:** D-014: "anyio cancel scopes must exit in the task that entered them; LangGraph nodes call tools from different tasks. Found via a real failure in tests." **Fix:** `McpClient` owns the connection in a dedicated task (`src/aeos/mcp_client.py`).

### 2.3 Docker `put_archive` refuses a read-only rootfs
**Label:** Verified historical issue. **Evidence:** D-017. **Fix:** stream tar through an in-container `tar -x` over exec stdin instead of weakening the hardening.

### 2.4 Cleanup could kill its own container; fork bomb outlived the command
**Label:** Verified historical issue. **Evidence:** D-018 ("Found by a failing test"), D-019 (children outlived the command, so no timeout fired). **Fix:** keepalive runs as a different uid; leftover-process check after every exec with destroy as fallback (`ExecResult.sandbox_destroyed`). Tests: `tests/sandbox/test_abuse.py::test_fork_bomb_is_contained_and_leaves_no_stragglers`.

### 2.5 Models confuse "previous commit touching the file" with "parent commit"
**Label:** Verified historical issue. **Evidence:** PROGRESS Phase 8 ("fired in 2 of 3 live runs", event `baseline_corrected`) and D-035. The baseline then predated `benchmark.py` and the verifier's baseline benchmark crashed. **Fix:** the harness computes `<suspect>~1` and rejects suspect SHAs that are not commits. **Residue:** the 3 Oct result file still shows `latency-serialization_quadratic-05` escalating with `python: can't open file '/workspace/benchmark.py'`, i.e. the problem was visible in the eval before the fix. **Lesson:** hard facts are derived by code, never reported by the model.

### 2.6 A wrong "last good" commit let the benchmark gate pass trivially
**Label:** Verified historical issue. **Evidence:** `tests/agents/test_verifier_gates.py::test_a_baseline_that_is_not_faster_than_the_head_cannot_verify_anything`, whose docstring says "Found by the live eval". Gate `regression_reproduced` (issue `baseline_not_faster`) exists because of it.

### 2.7 Stagnation detector flagged normal edit-run-edit
**Label:** Verified historical issue. **Evidence:** PROGRESS Phase 8, fix 2: mutating tools reset repeat counters (`ToolDef.mutates`).

### 2.8 First live eval: 50%, with infrastructure noise inside it
**Label:** Verified historical issue plus Observed limitation. **Evidence:** `evals/results/20261003T024031Z_live-gemini-full.md`: success 0.500 (15/30); three runs `crashed` with `TimeoutError`; one `escalated` with `analytics malformed: model produced no tool call 3 times (finish_reason='blocked')`; every rubric grade failed with `gemini error 404: This model models/gemini-2.5-pro is no longer available to new users`, giving rubric score 0.000. **Interpretation:** a mix of model errors (wrong commit blamed), provider-side issues (model withdrawn, a `blocked` finish) and harness limits (900 s timeout). Do not describe the 50% as purely "model quality".

### 2.9 Live eval results lost to a comparison crash
**Label:** Verified historical issue. **Evidence:** commit `2fab6fc` message: "A live run lost its results when comparison with an older file crashed on a metric that file predates." **Root cause from the diff:** `main_async` computed `compare(result["aggregate"], previous["aggregate"])` *before* writing the JSON. `compare` indexed `previous["overall"][key]` for every key in `METRICS`; the 3 Oct file had no `fix_rate`, and `metrics.py` in the first commit already defined it (`git show d9cf6ea:evals/metrics.py` contains `fix_rate`), so the old file was a schema older than the code. A `KeyError` aborted before any file was written. **Fix:** (a) write the JSON first with the comment "Save the runs FIRST: a live run costs real money"; (b) wrap comparison in `try/except Exception` that prints a warning; (c) `compare` skips metrics absent from either file (`continue`); (d) regression test `test_compare_with_an_older_results_file_skips_metrics_it_lacks` (docstring: "A live run crashed here once"). **Observed gap:** results are still held in memory until all jobs finish, so a crash mid-run would still lose finished tasks; and result files carry no explicit schema version.

### 2.10 Memory benefit not demonstrated
**Label:** Verified historical issue (an honest negative result). **Evidence:** PROGRESS Phase 8: memory-on LLM calls 65/17/19 vs memory-off 19/21/62 (3 trials per arm), "No benefit can be claimed"; D-036; README "a small comparison (3 runs per arm) showed no measurable benefit". Mechanism and safety are tested; benefit is not.

### 2.11 Cancel reuses the lease, not a kill
**Label:** Verified design decision. **Evidence:** D-047 and `tests/api/test_task_control.py`. `tasks.cancel` sets `cancelled` and clears the worker id; the running worker is fenced out at its next heartbeat. **Limit (Observed):** cancellation takes effect at the next heartbeat, not instantly (inference from the lease design).

### 2.12 Change requests: not every repo can be test-verified
**Label:** Observed limitation. **Evidence:** D-049, README "Known limitations": the offline sandbox runs only suites needing nothing installed (pytest, `node --test`); jest/mocha are reported "could not run", never as passing; a failing suite counts against the patch only if it passed on the unpatched checkout (`agents/verifier_change.py::run_repo_tests`). Honest reporting of "could not run" is the design.

### 2.13 Docker socket in the coding service
**Label:** Observed limitation (documented). **Evidence:** D-041. Anyone compromising that service controls the host Docker daemon. Mitigation suggestion in the decision: rootless Docker, sysbox or a dedicated sandbox host.

### 2.14 Tests share the dev database
**Label:** Observed limitation (documented). **Evidence:** README "Known limitations", CONTRIBUTING; `DELETE FROM aeos.tasks` in several fixtures.

### 2.15 CI never verified remotely; DB tests may skip there
**Label:** Observed limitation / Potential scenario. **Evidence:** PROGRESS Phase 0 "CI unverified remotely"; `ci.yml` has no Postgres service while `seeded_db` skips without it. See [22-testing-strategy.md](22-testing-strategy.md) section 7.

---

## 3. ROADMAP versus implemented

| ROADMAP item | State in the repo | Evidence |
|---|---|---|
| Phases 0 to 10 | Implemented and documented | PROGRESS.md |
| Phase 11 evals: dataset 30+, graders, metrics, repeated trials, results versioned, broken config lowers metrics | Implemented. CI job runs smoke eval: **not** in `ci.yml` (exists as pytest tests). "regression rate vs previous run" covered by `compare` | `evals/*`, `tests/evals` |
| Phase 12 failure injection: injector with config + API + dashboard button for MCP timeout, LLM rate limit, malformed output, DB down, sandbox crash...; fault table with recovery rate per type | **Only crash injection** (`AEOS_CRASH_AT`, `failure/crash.py`) plus targeted resilience tests; no fault table, no API/button | `src/aeos/failure/` has one module |
| Phase 13 security: adversarial test set, security eval table, JWT + RBAC (COULD) | Targeted tests only; PROMPT_INJECTION.md: "There is no large adversarial test set". Auth is optional scrypt/cookie login, not JWT/RBAC | `docs/PROMPT_INJECTION.md`, D-048 |
| Phase 14 dashboard | Implemented, well beyond "keep minimal" (trace waterfall, evals page, login, diff comments); no frontend tests | `web/`, commits `d663736`, `65a08b4` |
| Phase 15 docs/demo: README, ARCHITECTURE, DECISIONS, **DEMO, EVAL_RESULTS**, INTERVIEW_PREP, demo video | DEMO.md and EVAL_RESULTS.md do not exist (results are `evals/results/*.md` + dashboard); `make demo` exists; a promo video exists in `brag-output/` | `ls docs`, Makefile |
| Definition-of-done checklist | Boxes in ROADMAP section 8 are all unticked; PROGRESS.md has no entries for phases 11 to 15, dashboard, auth, cancel/retry | grep |
| Temporal (COULD) | Not adopted; D-033 | decisions |
| Kubernetes/OAuth (COULD) | Not present | tree |
| Real GitHub (COULD) | **Implemented** as a mirror inside the approved `create_pull_request` call (D-045), although PROGRESS Phase 2 still says "not built" | `mcp_servers/github/remote.py` |
| "Multi-agent: A2A real topology" | Implemented for regression tasks; README: change requests run in single-process mode, not over A2A | README |
| Provider: "two real providers" | Relaxed to one real (Gemini) + Fake (D-002) | decisions |

Docs lag (Observed): PROGRESS.md ends at the terminal client (11 Oct), so later work is documented only in commit messages, README and D-046 to D-050.

---

## 4. How to narrate the project honestly

Do:
- Lead with the design thesis in the ROADMAP: "deterministic code decides safety; LLMs advise" and show where it is enforced (policy, verifier, sandbox) with a test for each.
- Quote results with denominator, domain and caveats: "26 of 30 synthetic tasks, one trial each, about six points of standard error; earlier run was 15 of 30, and code, prompts and graders changed in between."
- Volunteer the negative results: memory showed no measurable benefit; first live eval had infrastructure noise.
- Use the real find-and-fix stories: the chaos suite's three bugs, the baseline-commit confusion, the lost eval results.
- Say "prototype": single worker, optional basic login (README's own words).

Do not:
- Claim a security control the code does not enforce (for example "prompt injection is solved"; the doc says the taint flag is coarse and there is no large adversarial set).
- Imply incremental development from the commit log, or that the five 14 Oct commits were separate sessions.
- Claim failure-injection coverage beyond crash and a handful of targeted tests.
- Present the heuristic rubric score (0.880 in every scripted run) as quality.
- Claim "exactly-once processing": it is at-least-once execution with idempotent effects.
- Present the improvement from 50% to 87% as caused by one change.

---

## Q-HIST-1. What were the hardest bugs you hit, and how did you find them?
**Difficulty:** Intermediate  **Topic:** Engineering narrative
**Short answer:** "The ones I can document: the chaos suite found three exactly-once bugs, such as a duplicate audit entry when a worker died between creating a PR and checkpointing. Live Gemini runs exposed that models confuse 'previous commit touching a file' with 'parent commit', which broke baselines, so the harness now derives the baseline itself. And a live eval exposed that a wrong baseline let the benchmark gate pass trivially, which added a regression-reproduced gate."
**Detailed explanation:** All three share a theme: they were found by running the real thing, not by reasoning. That is why the project prefers real MCP servers, Docker and Postgres in its integration tests.
**How this project uses it:** PROGRESS Phases 7 and 8; D-035; `test_a_baseline_that_is_not_faster_than_the_head_cannot_verify_anything`.
**Example/walkthrough:** Crash at `tool:create_pull_request:after`: PR made, checkpoint not written, worker B re-runs `publish_pr`; without a dedupe key the audit log would contain two `pr_created` rows.
**Follow-ups:**
- *Why not unit-test these?* The failure needs a process dying between two real side effects; a mock cannot represent it faithfully.
- *Could the model confusion recur?* Yes in other forms; the principle is to compute hard facts in code (D-035).
**Trade-offs and alternatives:** Formal model checking of the idempotency protocol would be stronger but heavy.
**Common interview mistake:** Inventing dramatic outages. Stick to what the repo records.
**Related:** [10-error-handling-and-reliability.md](10-error-handling-and-reliability.md), Q-TEST-2

## Q-HIST-2. Your live eval lost its results once. Walk me through root cause and fix.
**Difficulty:** Intermediate  **Topic:** Post-mortem
**Short answer:** "Comparison ran before saving. The older results file lacked a metric the newer code computes, so indexing it raised and the paid run vanished. I now write the JSON first, comparison is in a try/except, compare skips metrics missing from either side, and a test pins it."
**Detailed explanation:** Section 2.9 has the diff-level explanation. The deeper cause was schema drift between result files without a version field, combined with ordering that put an optional step ahead of a mandatory save.
**How this project uses it:** `evals/run_evals.py::main_async`, `evals/metrics.py::compare`, `tests/evals/test_evals_unit.py`.
**Example/walkthrough:** Old file: no `fix_rate` key; `current["overall"]["fix_rate"]` fine, `previous["overall"]["fix_rate"]` raises `KeyError`.
**Follow-ups:**
- *What is still exposed?* In-memory accumulation until all tasks finish, and no schema version. Per-task write-ahead would fix both. (Recommendation.)
- *Was the run reconstructed?* The 12 Oct run was added in the same commit; I cannot tell from the repo whether it is the re-run.
**Trade-offs and alternatives:** Append-only JSONL per task; versioned result schema.
**Common interview mistake:** Blaming "a typo". It was an ordering and compatibility flaw.
**Related:** Q-EVAL-6

## Q-HIST-3. How did the system improve from 50% to 87%, and can you prove why?
**Difficulty:** Advanced  **Topic:** Honest attribution
**Short answer:** "I cannot prove why. The runs are 3 Oct (15/30) and 12 Oct (26/30), and the code, prompt and grader hashes all differ, plus the judge was broken in the first run and three tasks timed out. The README lists candidate causes, blame lookup, baseline selection, stronger planning model, but it is not an ablation."
**Detailed explanation:** To attribute, you would hold everything but one change fixed, run several trials per task, and compare with a confidence interval. The harness supports it (`--trials`, version hashes), it just has not been done.
**How this project uses it:** `evals/results/*.md` meta lines; commit `2fab6fc` message.
**Example/walkthrough:** Compare hashes: code `61faa5ca5e9d` to `02fe54dfda03`, prompts `9866d65c165c` to `e8919857ce1f`, graders `2b09da2e678a` to `44f262cacf55`, dataset and policy unchanged.
**Follow-ups:**
- *Is part of the gain grader-related?* Possibly: the grader hash changed and `fix_rate` and the working judge appear only in the later run.
- *What is the cheapest next experiment?* Re-run the Oct 12 code with the old prompts (or vice versa), 3 trials.
**Trade-offs and alternatives:** Ablations cost real money; at about $0.065 per task a 30-task, 3-trial run is on the order of $6 using the quoted per-task cost (arithmetic, not a measured total).
**Common interview mistake:** Claiming credit for a specific fix.
**Related:** Q-EVAL-1

## Q-HIST-4. What did you plan and not build?
**Difficulty:** Intermediate  **Topic:** Scope management
**Short answer:** "Phase 12's fault injector with a dashboard button and a fault-to-recovery table, Phase 13's adversarial prompt-injection set and JWT/RBAC, a DEMO and EVAL_RESULTS doc, Temporal, and Kubernetes. Crash injection exists and is thorough; other faults are covered by targeted tests rather than an injector. I documented these gaps instead of hiding them."
**Detailed explanation:** The ROADMAP's own cut order put Kubernetes, OAuth and Temporal first. It also defined 'never cut' items (verifier loop, exact-resume HITL, durable execution, sandbox, evals, tracing); all are present. See the table in section 3.
**How this project uses it:** `src/aeos/failure/crash.py` only; `docs/PROMPT_INJECTION.md` "What is not covered".
**Example/walkthrough:** Resilience covered without an injector: A2A stall and deadline tests, flaky specialist retried (`test_replan_e2e.py`), `McpClient` retry rules.
**Follow-ups:**
- *What would the fault table look like?* Rows: MCP timeout, LLM 429, malformed tool output, Postgres down, sandbox crash; columns: detection, recovery, pass/fail. (Recommendation.)
**Trade-offs and alternatives:** Depth on crash recovery over breadth of fault types.
**Common interview mistake:** Saying "everything in the roadmap is done".
**Related:** [02-repository-tour.md](02-repository-tour.md)

## Q-HIST-5. How do you explain the git history of this project?
**Difficulty:** Beginner  **Topic:** Transparency
**Short answer:** "The history is eight snapshot commits, not a play-by-play: the core system arrived in one commit on 4 Oct, then the terminal client, dashboard, auth, change-request tests and evals fixes. The detailed phase-by-phase record is in PROGRESS.md and DECISIONS.md, and the earliest eval runs predate the first commit."
**Detailed explanation:** An interviewer may check `git log`. Pre-empt it: the log shows when work was committed, not when it was done. Five commits share one timestamp.
**How this project uses it:** section 1.
**Example/walkthrough:** `d9cf6ea` touches 245 files (35,052 insertions); `2fab6fc` touches five.
**Follow-ups:**
- *Can you reproduce the 3 Oct result from git?* No: the code that produced it is not a commit.
**Trade-offs and alternatives:** Commit per phase would have made hash-to-result mapping reproducible; the version hashes in result files partly compensate.
**Common interview mistake:** Defending the history as "incremental".
**Related:** [01-project-overview.md](01-project-overview.md)
