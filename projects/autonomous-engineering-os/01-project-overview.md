# 01 - Project Overview (AEOS: Autonomous Engineering OS)

**What this file covers.** What AEOS is, who it is for, what it can and cannot do, the two task types it supports, why the design is the way it is, and how to pitch it in an interview without overclaiming. Deep dives live in the sibling files: [03-system-architecture](03-system-architecture.md), [04-end-to-end-workflows](04-end-to-end-workflows.md), [07-state-durability-and-persistence](07-state-durability-and-persistence.md), [10-error-handling-and-reliability](10-error-handling-and-reliability.md). Labels used: **Verified** (read in code/tests/commits), **Inferred rationale**, **Potential failure scenario**, **Recommendation**.

---

## 1. One-paragraph description (Verified: README.md, graph/build.py)

AEOS takes an engineering request in plain English ("update the `ws` dependency and raise a PR", or "find why checkout latency regressed after the last deploy, fix it, prepare a PR"), turns it into a plan, runs specialist LLM agents (research, analytics, coding) that use real tools, applies the change inside a locked-down Docker sandbox, **re-verifies the patch independently of the agent that wrote it**, **pauses for a human approval**, and only then opens a pull request. It never merges. State is checkpointed in Postgres so a worker killed with `kill -9` is resumed by another worker without duplicating side effects.

The one-line version: *a LangGraph state machine in which models propose and deterministic code disposes.*

## 2. Who uses it and what for

| User | What they do | Where |
|---|---|---|
| Engineer / reviewer (the primary "human in the loop") | Submits a request, reads the evidence card (diff, gate results, tests/benchmark, judge concerns), approves or rejects with feedback | Dashboard (`web/`), terminal client (`src/aeos/client/`), or `aeos approve/reject` |
| Operator | Runs workers, watches traces, cancels or re-runs tasks | `aeos serve`, `aeos worker`, `POST /tasks/{id}/cancel`, `/retry`, Langfuse |
| The repo owner (you, in an interview) | Explains design trade-offs: agents vs workflow, durability, safety boundary | This knowledge base |

**Inferred rationale:** the project is a portfolio/learning system, not a product with customers. README claims one live run ("ws 8.18.3 -> 8.22.0, 17 LLM calls, 79k tokens, about $0.03, 65 s"); I did not verify that number, it is the author's README claim. Do not present it as a benchmark; present it as one demonstration.

## 3. Task types (Verified: `core/state.py` `TaskKind`, `graph/nodes.py`, `graph/routing.py`)

| Aspect | `change` (change request) | `regression` (latency regression) |
|---|---|---|
| Target | The real GitHub repo configured in `AEOS_GITHUB_REPO`, mirrored to a local bare repo (`runtime.refresh_target`, `target.sync_mirror`) | A seeded demo repo + fake telemetry in Postgres (`demo_target/`) |
| Plan | `research -> coding -> verifier -> publish` (analytics forbidden: `supervisor.validate_steps`) | `analytics + research (parallel) -> synthesize -> coding -> verifier -> publish` |
| Extra node | none | `synthesize`: LLM picks the root cause from both agents' findings |
| Hard facts checked by code | gates in `verifier_change.run_change_gates` | `GraphNodes._check_research` forces the suspect to be a real commit and the baseline to be its nearest benchmarkable ancestor (`Runtime.baseline_for`); benchmark vs baseline in `verifier.run_gates` |
| Memory | not used (`reflect` returns `{}` for `change`) | recall at plan time, reflect after finish |
| Publish target | real GitHub (when `AEOS_GITHUB_PUBLISH` is on for the target server) | local PR store only (`runtime.open_runtime` forces `github_publish=False` for the demo repo) |

The default kind when the API is asked without one is `change` if `settings.target_ready` (repo + token configured) else `regression` (`api/app.py create_task`).

## 4. Feature inventory (what actually exists)

Verified present in code:

- **Planning and replanning** by an LLM with deterministic validation and fallback (`agents/supervisor.py`: `validate_steps`, `finalize_plan`, `fallback_replan_steps`). A verifier step and a publish step are always appended by code, never by the model.
- **Parallel specialists** inside one graph node (`GraphNodes.dispatch` uses `asyncio.gather`).
- **Independent verifier** with hard gates + advisory LLM judge that can only make a verdict stricter (`GraphNodes.verify`, `agents/judge.apply_judgement`). See [13-verification-and-risk-policy](13-verification-and-risk-policy.md).
- **Risk policy** from `policy/risk_policy.yaml`: deny always wins, default is `require_approval`, merging/deleting is denied, branch writes only to `aeos/*`.
- **Human approval** as a graph `interrupt()`, with expiry (default 24 h), exactly-once decisions (compare-and-set), arguments-hash binding (`hitl.py`, `store/approvals.py`, `graph/publish.py`).
- **Durable execution**: Postgres checkpointer + a lease-based task queue + idempotency keys (`graph/checkpoint.py`, `store/tasks.py`, `worker.py`). See [07](07-state-durability-and-persistence.md).
- **Crash testing**: `failure/crash.py` injects SIGKILL at named points; `tests/chaos/test_chaos.py` kills a real worker at every node boundary and several tool points.
- **Observability**: one OTel trace per task rejoined across nodes/workers (`TaskState.trace`), events persisted to `aeos.events` and streamed over SSE. See [20](20-observability-and-telemetry.md).
- **MCP tool servers, A2A specialist services, sandbox, memory, API, dashboard, evals**: covered in siblings 15, 16, 12, 08, 18, 19, 23.

Not present (do not claim): Redis is configured (`Settings.redis_url`) and in docker-compose, but **no code in `src/` uses it** (grep: only `core/config.py`). Docs say "reserved for queues/caches". There is no horizontal autoscaling logic, no multi-tenant isolation, no automatic merge, no rollback of an opened PR.

## 5. Value proposition and honest framing

**Inferred rationale for the design:**

1. *Autonomy is only useful if it is safe to leave alone.* The architecture spends most of its code on the parts that make unattended runs safe (sandbox, verifier, policy, approval, idempotency), not on prompts.
2. *Separate the author from the checker.* The coding agent claims success; the verifier recomputes everything on a fresh checkout. This is the same reason code review exists.
3. *Make the LLM's role narrow and checkable.* Plans are validated; research output is corrected by code (`_check_research`); judge cannot override gates.
4. *Make failure a first-class path.* Replan, escalate, expire, cancel are all routed states, not exceptions.

**What it is not:** a general coding agent that can do anything. Its scope is bounded to the two task types, to a sandboxed single-repo checkout, and to demo-scale data. The latency regression world is synthetic (seeded), which is great for evals (known ground truth) but means "it found a real production regression" is not a claim you can make.

## 6. How to pitch it

**30 seconds.** "AEOS is a multi-agent system that takes an engineering request, researches the repo, makes the change in a no-network Docker sandbox, verifies the patch independently of the agent that wrote it, waits for a human to approve, and then opens a real GitHub PR. The interesting part is not the agents, it's the guardrails: a deterministic risk policy, an independent verifier, approval bound to the exact PR arguments, and Postgres-backed durability so a `kill -9` mid-run resumes without duplicating the PR."

**2 minutes.** Add: LangGraph graph with hand-written routing (`graph/routing.py`, pure functions, unit tested); two task types; verifier gates + advisory judge; leases + idempotency keys; chaos tests that kill the worker at every node boundary; replan loop that is forced to change strategy on the second failure; what you would do differently (Section 7).

**Lead with trade-offs, not features.** Examples that show judgement: re-run the interrupted node instead of checkpointing mid-node (DECISIONS D-032); LangGraph + own lease queue rather than Temporal (D-033); LLM output validated and given deterministic fallbacks.

## 7. Known limits you should volunteer (Verified unless marked)

- Crash recovery re-executes the interrupted node, so LLM work in that node is repeated and its budget spend is not counted (budget is checkpointed at node end). **Potential failure scenario**, see [10](10-error-handling-and-reliability.md).
- Budgets are only checked inside the agent loop, not around planner/synthesis/replan/judge calls.
- No graceful drain on shutdown: the worker CLI installs no signal handler, so SIGTERM behaves like a crash and recovery relies on lease expiry.
- Events can be duplicated after a replay and lost on a hard kill (in-memory queue in `DbSink`).
- The Docker-socket mount for the coding service is an accepted risk (docs/ARCHITECTURE.md section 5).

---

## Interview questions

## Q-OVR-1. In two minutes, what is AEOS and why would anyone trust it?
**Difficulty:** Beginner  **Topic:** Project framing
**Short answer:** It turns a plain-English engineering request into a verified, human-approved GitHub PR. Trust comes from structure, not from the model: the agent's patch is re-checked by a separate verifier on a fresh checkout, risky actions go through a deterministic policy, a human approves the exact PR arguments, and the run is durable and traced.
**Detailed explanation:** An LLM agent loop is probabilistic; if the same component writes the patch and declares success, you inherit its blind spots. AEOS splits roles: specialists propose, `GraphNodes.verify` recomputes, the policy engine gates side effects, and a human is the final authority. Everything the LLM could get wrong in a way that matters (baseline commit, suspect commit, verdict) is overruled or recomputed by code.
**How this project uses it:** `graph/build.py` assembles the graph; `graph/nodes.py` implements plan/dispatch/verify/replan/finish; `graph/publish.py` holds the three-node publish sequence; `policy/risk_policy.yaml` is the rule file.
**Example/walkthrough:** "Update ws" -> plan(research, coding, verifier, publish) -> research reads package files -> coding edits package.json/lockfile in sandbox -> verifier re-applies patch to fresh checkout and checks safety/syntax/lockfile agreement -> approval card -> approve -> branch, commit, PR.
**Follow-ups:**
- *What stops it merging?* The policy rule `never-merge` denies `merge_pull_request`, `delete_branch`, `force_push` etc., and no merge tool is exposed by the github MCP server (server.py has create_branch, commit_patch, create_pull_request and reads). Verified.
- *Is the approval just a button?* No: it stores `arguments_hash` of the PR arguments; at `create_pr` a grant bound to that hash is attached and the policy blocks a mismatch (`tests/graph/test_hitl.py::test_tampering_with_the_approved_arguments_is_blocked`). Note this protects against tampering with stored arguments, not against a human approving a bad diff.
**Trade-offs and alternatives:** A fully autonomous merge would be faster but unsafe; a pure workflow (no LLM planning) would be more predictable but couldn't adapt after a failed verification.
**Common interview mistake:** Saying "the verifier is an LLM that double-checks". The gates are deterministic; the LLM judge is advisory and can only tighten.
**Related:** [13-verification-and-risk-policy](13-verification-and-risk-policy.md), [03-system-architecture](03-system-architecture.md).

## Q-OVR-2. What are the two task types and why do they differ?
**Difficulty:** Beginner  **Topic:** Scope
**Short answer:** `change` works on a real GitHub repo (research then coding); `regression` works on a seeded demo service with synthetic telemetry (analytics + research, then synthesis, then coding). Regression exists because it has known ground truth for evals and exercises the full multi-agent loop.
**Detailed explanation:** The `kind` field on `TaskState` changes planning prompts (`PLANNER_SYSTEM_CHANGE` forbids analytics), the specialist set (`rt.change_specialists` vs `rt.specialists`), the verifier (`run_change_gates` vs `run_gates`), routing (`synthesize` only for regression), and whether memory is used.
**How this project uses it:** `GraphNodes.dispatch` picks `specialists = change_specialists if kind == "change" else specialists`; `route()` checks `state.kind == "regression"` before `synthesize`.
**Example/walkthrough:** regression: s1 analytics, s2 research run in one `asyncio.gather`; `route` sees `investigation_done` and `root_cause is None` -> `synthesize`.
**Follow-ups:**
- *Why not make the demo repo the only target?* A synthetic world cannot prove the system works on real repositories; the change path produces real PRs.
- *Why is memory regression-only?* `reflect` early-returns for `change` ("memory only models regression episodes"), Verified in `nodes.py`.
**Trade-offs and alternatives:** One generic task type would be simpler to explain but would blur which checks apply.
**Common interview mistake:** Claiming it diagnoses real production incidents; the telemetry is seeded.
**Related:** [05-agents-and-orchestration](05-agents-and-orchestration.md), [04-end-to-end-workflows](04-end-to-end-workflows.md).

## Q-OVR-3. What problem does AEOS solve that a plain "coding agent" does not?
**Difficulty:** Intermediate  **Topic:** Positioning
**Short answer:** The problem is trust and operability of unattended agents: independent verification, policy-gated side effects, human approval with evidence, crash-safe resumption, and a full audit/trace. A bare coding agent gives you a diff and hope.
**Detailed explanation:** The system treats "agent said done" as a claim. `verify` re-applies the patch; `route` will not publish unless the verifier step is `done`; `publish_prepare` writes an `ApprovalRequest` with evidence (gates, tests, benchmark, judge, patch) before anything visible happens.
**How this project uses it:** `graph/routing.py::route` (publish only if `verifier_done and publishing`), `graph/publish.py::PublishNodes.prepare`.
**Example/walkthrough:** A coding agent that writes nothing is failed by the verifier (`tests/graph/test_e2e.py::test_coding_agent_that_writes_nothing_is_failed_by_the_verifier`).
**Follow-ups:**
- *What is not solved?* Semantic correctness beyond the available checks (a change request repo without tests only gets syntax/safety gates and the PR body says "no repository tests were run", `pr_body_change`).
**Trade-offs and alternatives:** More checks = slower and costlier runs.
**Common interview mistake:** Overstating verification: for repos with no tests it is weaker, and the code says so in the PR body.
**Related:** [13](13-verification-and-risk-policy.md), [22-testing-strategy](22-testing-strategy.md).

## Q-OVR-4. Which parts are LLM-driven and which are deliberately not?
**Difficulty:** Intermediate  **Topic:** Agent vs deterministic boundary
**Short answer:** LLM: planning/replanning, specialist reasoning and tool choice, root-cause synthesis, patch writing, an advisory judge, memory curation. Deterministic: routing, retry/escalation, risk policy, sandbox limits, verifier gates, baseline computation, idempotency, approval binding, budgets.
**Detailed explanation:** Rule of thumb in the code: anything that can cause a side effect or decides pass/fail is code. Even LLM outputs go through validators with fallbacks (`create_plan` falls back to `default_plan_steps` on invalid plans or `LLMError`).
**How this project uses it:** `core/retry.py` (pure policy), `graph/routing.py` (pure functions), `core/risk.py`, `core/budget.py`.
**Example/walkthrough:** `test_planner_outage_falls_back_to_the_default_plan` shows planner failure does not fail the task.
**Follow-ups:**
- *Where does an LLM decision still influence safety?* The coding agent's patch content; mitigated by sandbox + verifier gates + human review.
**Trade-offs and alternatives:** Pushing more into code reduces flexibility; pushing more into prompts increases variance.
**Common interview mistake:** "Prompts enforce the policy." They do not; `ToolRuntime.execute` calls `gate.decide` before every tool.
**Related:** [05](05-agents-and-orchestration.md), [14-security-and-prompt-injection](14-security-and-prompt-injection.md).

## Q-OVR-5. How would you demo it, and what would you say if it fails live?
**Difficulty:** Intermediate  **Topic:** Demo strategy
**Short answer:** Use the scripted `wrong_fix` replay (no API key): the first fix fails verification, the supervisor replans and the second passes, then approve. If something fails, point to the failure path: it should show up as `step_retry`/`escalated` with an error and be resumable, which is itself the feature.
**Detailed explanation:** README documents `make demo`; replays script only the model's answers while tools, sandbox and verifier are real. A live Gemini run is non-deterministic.
**How this project uses it:** `llm/fake.py` (scripted LLMs), `demo/trajectories.py`, `AEOS_DASHBOARD_REPLAY`.
**Example/walkthrough:** README states the recorded GIF's retry segment is illustrative; say so if asked.
**Follow-ups:**
- *Is the replay honest?* The README labels it; do not imply the model produced those answers live.
**Trade-offs and alternatives:** Live runs show real behaviour but risk variance and cost.
**Common interview mistake:** Presenting the replay as a live autonomous run.
**Related:** [23-evaluation-framework](23-evaluation-framework.md), [22](22-testing-strategy.md).

## Q-OVR-6. What would you improve first, and why?
**Difficulty:** Advanced  **Topic:** Self-critique
**Short answer:** (1) Count budget spend durably across crashes and bound planner/judge calls; (2) graceful drain on SIGTERM; (3) make events replay-safe; (4) separate the "claim count" from "crash count" in the task queue. All are small, local changes derived from reading the code.
**Detailed explanation:** See [10](10-error-handling-and-reliability.md) sections on gaps. Each is a **Recommendation**, not an existing feature.
**How this project uses it:** `core/budget.py`, `worker.py`, `store/events.py`, `store/tasks.claim`.
**Example/walkthrough:** A task that goes through three approval rounds consumes three `attempts` increments even with zero crashes because `claim` always increments.
**Follow-ups:**
- *Why not Temporal?* DECISIONS D-033: the chaos suite passes on LangGraph + leases; Temporal adds an operational dependency. Revisit at much larger scale.
**Trade-offs and alternatives:** Each fix adds complexity; the current design is intentionally simple.
**Common interview mistake:** Listing generic "add more tests/monitoring" instead of specific code-grounded gaps.
**Related:** [24-technical-challenges-and-history](24-technical-challenges-and-history.md), [07](07-state-durability-and-persistence.md).
