# 10 - Error Handling and Reliability

**What this file covers.** Every layer where AEOS can fail and what it does about it: LLM retries, agent-loop limits, MCP transport retries, step retries, replans, escalation, budgets, timeouts, partial failure, worker crashes, shutdown, and a candid "implemented vs not implemented" list. Related: [04-end-to-end-workflows](04-end-to-end-workflows.md), [07-state-durability-and-persistence](07-state-durability-and-persistence.md), [06-llm-integration](06-llm-integration.md), [12-sandbox-and-execution-safety](12-sandbox-and-execution-safety.md), [22-testing-strategy](22-testing-strategy.md).

Labels: **Verified** (code/tests), **Inferred rationale**, **Potential failure scenario**, **Recommendation**.

---

## 1. First principles

Failures differ in *who can fix them*: transient (retry), wrong output (re-plan), impossible (escalate to a human), or fatal to the process (resume elsewhere). A reliable system names each class, bounds every retry loop, and has a final owner (a human) for what automation cannot resolve. AEOS encodes this as layered, bounded loops, from innermost to outermost:

| Layer | Loop | Bound | Code |
|---|---|---|---|
| 1. LLM call | retry transient API errors with backoff + jitter | `max_retries=4` | `llm/gemini.py` |
| 2. Structured output | re-ask on invalid/missing `submit` | `retries=1` | `llm/structured.py::complete_structured` |
| 3. Agent loop | malformed turns, invalid `finish`, stagnation, step cap | `max_malformed=3`, `finish_retries=2`, stagnation at 3 warns / 5 stops, `max_steps=25` | `agents/loop.py` |
| 4. MCP call | retry transport failures/timeouts for safe calls | `max_retries=2`, backoff `0.2*2^n`, timeout 30 s | `mcp_client.py` |
| 5. Step | retry the specialist step | `max_step_attempts=2` | `core/retry.py`, `GraphNodes.dispatch` |
| 6. Plan | replan after failed verification / human rejection | escalate at 3 | `core/retry.py`, `GraphNodes.replan` |
| 7. Task run | requeue after unexpected exception or crash | `max_attempts=5` claims | `worker.py`, `tasks.claim` |
| 8. Human | escalated tasks wait for a person | n/a | status `escalated` |

## 2. Details by layer (Verified)

**LLM (Gemini).** Transient = HTTP 429/500/502/503/504, `TimeoutError`, `ConnectionError`, or an *unusable turn* (no tool call and no content with a retryable finish reason). Delay `min(2**attempt, 20) + uniform(0, 0.5)` seconds (the unusable-turn case sleeps `0.5*(attempt+1)`). Non-transient API errors raise `LLMError` immediately; exhaustion raises `LLMTransientError`. I found no explicit per-request timeout setting in this provider code; **Potential failure scenario:** a hung request is bounded only by the SDK default.

**Structured output.** Forces a `submit` tool call whose args validate against a pydantic model; on `ValidationError` the error text is fed back once. Failure raises `StructuredOutputError` (an `LLMError`).

**What callers do with LLM failure:**

| Caller | On `LLMError` | Result |
|---|---|---|
| `create_plan` | fall back to `default_plan_steps` | task continues (`test_planner_outage_falls_back_to_the_default_plan`) |
| `replan` (supervisor) | deterministic `fallback_replan_steps`, reason recorded | task continues |
| `synthesize_root_cause` in node | `except Exception` -> `status=failed` | task ends, no retry (gap) |
| Agent loop | `LoopResult("error")` | step failure -> step retry policy |
| `judge_patch` | not reviewed in this scope | see [13](13-verification-and-risk-policy.md) |

**Agent loop limits.** The loop never raises for model/tool misbehaviour; it returns a status (`finished, max_steps, budget, stagnated, malformed, error`). Stagnation counts identical (tool, args) calls; a mutating tool (`ToolDef.mutates`) clears the counter so re-running checks after an edit is "progress". Old tool results are elided when context exceeds `max_context_chars` (180k). Tool exceptions are caught in `ToolRuntime.execute` (`[tool_crashed]`) and returned to the model as `<tool_error>`.

**MCP client retries (safe-only).** `_retryable`: tool risk is `read`, or the call has a non-empty `idempotency_key`. Everything else (non-idempotent writes) gets exactly one attempt. Tool-level errors (`isError`) are never retried; they become `McpToolError(code, message)` and then a `<tool_error>` for the model. After retries: `McpTransportError`.

**Step retry (`dispatch`).** For each non-finished specialist result:
- `res.status == "budget"` -> always `escalate` (never retried);
- else `RetryPolicy.on_step_failure(step.attempts)`: attempts 1 -> `retry` (step back to `pending`, event `step_retry`), attempts >= 2 -> `step.status=failed`, task `escalated` with "needs human attention".
- A raised exception inside a specialist is converted to `SpecialistResult(status="error")` in `run_one`, so one specialist crash does not abort the others or the node.

**Replan.** `RetryPolicy.on_verification_failure(failures)`: `1 -> replan`, `2 -> replan_new_strategy`, `>=3 -> escalate`. `failures` includes human rejections. Both thresholds are fields on a frozen dataclass and are unit-tested (`tests/core/test_retry.py`).

**Escalation** is a first-class outcome: `status="escalated"` routes to `finish`, `build_report` records the error, the dashboard shows needs-attention. Escalation causes (Verified): step failed twice; budget; 3rd failed verification/rejection; publish denied by policy; publish tool error; approval expired; crash-loop guard in `tasks.claim`; "research did not provide a last-good commit".

**Budgets (`core/state.Budget`, `core/budget.BudgetTracker`).** Four limits: `max_steps=200`, `max_tokens=1_500_000`, `max_cost_usd=5.0`, `max_wall_seconds=1800`. `Budget.exhausted()` returns a reason string. One tracker per node (`BudgetTracker(state.budget)` copies the budget), shared by parallel specialists in that node, and its result is returned as `state.budget` at node end. In A2A mode remote usage is added to the supervisor tracker after a delegation completes.

Gaps (all **Verified by grep: `exhausted` is called only in `agents/loop.py`**):
1. Only the agent loop checks the budget, once per iteration. Planner, synthesis, replan, judge, memory calls add usage but are never gated; the verifier and publish nodes do not check at all.
2. The wall-clock limit is measured from `started_at` (first tracker, in `plan`), so time spent **waiting for human approval** counts. **Potential failure scenario:** a task that waits more than 30 minutes for approval, is rejected, and replans will find the wall-clock budget exhausted in the first agent-loop iteration of the new coding step (`status="budget"` -> escalate). The default approval TTL is 24 h, so this is likely for real use unless `max_wall_seconds` is raised. The token/cost budgets are not affected.
3. Parallel agents can overshoot (`test_budget_exhaustion_halts_the_whole_task` asserts `steps_used <= 8` for `max_steps=4`: "parallel agents may each finish their in-flight step").
4. Spend inside a node that crashes is lost (see [07](07-state-durability-and-persistence.md)).
5. Per-task cap only; no global/daily spend cap. `max_cost_usd` is settable per task via the API (`TaskRequest.max_cost_usd`, validated `0 < x <= 50`).

**Timeouts (Verified values).**

| Where | Value | Source |
|---|---|---|
| MCP call | 30 s default | `McpClient(default_timeout=30.0)` |
| Sandbox command | `exec_timeout_s=60`, wrapped in `timeout --signal=KILL`; host-side future timeout `limit+5` | `sandbox/policy.Limits`, `sandbox/manager.exec` |
| Sandbox lifetime | `ttl_s=1800` for reaper TTL | `Limits`, `sandbox/reaper.py` |
| A2A whole delegation | `a2a_timeout_s=900` | `Settings`, `A2ASpecialist.run` |
| A2A silence | `a2a_stall_s=120` read timeout (a silent service is treated as dead) | `A2ASpecialist._delegate` |
| A2A connect | 10 s | `httpx.Timeout(10.0, read=stall_s)` |
| A2A cancel | 5 s best effort | `_best_effort_cancel` |
| Lease | 30 s, heartbeat 10 s | `Worker` |
| Approval TTL | 24 h | `Settings.approval_ttl_s` |
| Task wall clock | 1800 s (see gap above) | `Budget` |
| LangGraph recursion | 120 supersteps | `_config` |

## 3. Failure table

| Failure | Detected by | Behaviour | Tested? |
|---|---|---|---|
| Gemini 429/5xx | exception code | backoff retry x4 then `LLMTransientError` -> step error -> step retry | LLM layer not exercised against live API in default tests (`live` marker) |
| Planner outage | `LLMError` | default plan | yes |
| Invalid LLM plan | `validate_steps` | default plan with reason | yes (e2e) |
| Specialist loops/garbage | loop status | step retry once, then escalate | yes (`test_a_failing_specialist_fails_the_task_loudly`, `test_a_flaky_specialist_is_retried...`) |
| Hallucinated suspect commit | `_check_research` | treated as step error, retried | yes |
| Specialist raises | `run_one` try/except | `SpecialistResult(error)` | via above |
| A2A endpoint dead/slow | exception/stall/deadline | failover to next endpoint, else step error; remote cancel | A2A tests in `tests/a2a` (not in my scope) |
| MCP tool error | `isError` | error text to model, not retried | tools tests |
| MCP transport hiccup | exception | retry if read or keyed | `tests/mcp` |
| Patch wrong | verifier gates | replan ladder | yes |
| Node raises unexpectedly | `Worker.run_once` except | `release_with_error`: back to `queued`, `last_error` set; next claim resumes checkpoint | indirectly |
| Worker `kill -9` | lease expiry | another worker resumes | chaos suite |
| Repeated worker deaths | `attempts >= max_attempts` | `escalated` | yes (store test) |
| Approval ignored 24 h | `expire_due` in `housekeeping` | request `expired`, task woken with expired payload, escalates | yes |
| Late approval | `decide` | HTTP 410 | yes |
| Double approval | CAS | second gets 409 | yes |
| Policy denies publish | `gate.decide` | `escalated` | yes |
| Cancel | API | fenced at next heartbeat | store-level only |
| Budget exhausted | `exhausted()` in loop | `status=budget` -> escalate | yes |
| Target repo unreachable at plan | exception in `refresh_target` | `failed` with message | not seen |
| Postgres down | psycopg exceptions | no explicit handling: node/worker exceptions; worker loop `run_once` does not catch exceptions from `tasks.claim` (**Potential failure scenario**: `run_forever` would propagate and the process exits; compose `restart: unless-stopped` for `api` brings it back) | no |

## 4. Partial failure

- **Inside `dispatch`:** specialists run concurrently; each produces its own result; successes mark their step `done` and update artifacts even if another failed (the loop over `zip(ready, results)` handles each independently). Only the failing step is retried on the next `dispatch`. Because the *node* returns once, a crash loses all of them.
- **Publish:** `create_branch` and `commit_patch` may succeed, the approval may be pending; if the run is later rejected, the **branch remains** on the remote. Nothing deletes it (the policy denies `delete_branch`). Next attempt uses `aeos/<task>-v<n+1>`. **Potential failure scenario:** orphan `aeos/*` branches accumulate in the target repo. Recommendation: periodic cleanup by a human or a dedicated policy-approved job.
- **PR creation:** local PR row first, GitHub mirror after; a crash between leaves a row without `remote_url`, repaired on replay (`_mirror` is retried).
- **Memory/`reflect`:** failures are swallowed by design (`reflect` try/except; recall try/except emits `memory_error`) so memory never fails a task (`test_memory_failure_never_fails_the_task`).
- **Sandbox leftovers:** `Worker.housekeeping` every `sweep_s`: `approvals.expire_due()` then `sandbox.reap(active_tasks=active_task_ids, grace_s=reap_grace_s)`; `run_once` calls `reap_task` after any non-paused finish. The chaos helper asserts no containers labelled `aeos.task_id=<id>` remain.
- **Event/telemetry failure:** swallowed so the run continues (DbSink); consequence is missing timeline entries.

## 5. Graceful shutdown

**Verified: there is no graceful drain.**
- `cli.py::_worker` creates `stop = asyncio.Event()` but nothing sets it: no `signal` handler is installed anywhere in `src/aeos` for the worker (grep for `signal` hits only `failure/crash.py`). `run_forever(stop)` can therefore only end by exception or process death. SIGINT surfaces as `KeyboardInterrupt` -> exit code 130; SIGTERM kills with default disposition.
- `api/app.py` lifespan: on shutdown `ctx.stop.set()`, then `ctx.worker_task.cancel()` with the comment "like a crashed worker: its lease will expire". So in-flight tasks are abandoned deliberately and recovered by another claim after up to `lease_s`.
- `open_runtime`'s `AsyncExitStack` closes MCP child processes; sandboxes of an abandoned task are left for the reaper.

This is coherent *because* recovery is the primary design (crash = normal), but it means every deploy restarts in-flight nodes (repeated LLM spend) and delays tasks by the lease. **Recommendation:** on SIGTERM stop claiming, let the current node finish up to a deadline, then `release` the lease explicitly (`status='queued'`, `worker_id=NULL`) so recovery is immediate.

## 6. Implemented vs not implemented

| Implemented (Verified) | Not implemented / partial |
|---|---|
| Bounded layered retries; no unbounded loops | Circuit breaker around Gemini or MCP servers |
| Deterministic retry/replan/escalation policy with unit tests | Per-error-type backoff at step level (step retry is immediate) |
| Crash recovery by lease + checkpoint; chaos-tested at 16 node points + 7 tool points + random kills | Graceful drain / SIGTERM handling |
| Exactly-once effects via idempotency keys | Durable budget accounting across crashes |
| Approval expiry, exactly-once decisions | Budget checks outside the agent loop; wall-clock excludes human wait |
| Crash-loop guard | Dead-letter queue beyond the `escalated` status; no alerting hook |
| Sandbox reaping and TTL | Cleanup of orphaned remote branches |
| Failure paths surfaced to the user (`error`, events, report) | Rate limiting / global spend cap |
| Cancellation (cooperative) | Pre-flight "is the task still active" check before irreversible calls |

---

## Interview questions

## Q-REL-1. Describe the retry layers and why they are separate.
**Difficulty:** Intermediate  **Topic:** Retry design
**Short answer:** LLM transport retries (4x with backoff), structured-output re-ask (1x), agent-loop self-correction (3 malformed, 2 finish retries), MCP retries (2x, only safe calls), step retry (1 retry), replan (up to 2 replans then escalate), and worker requeue (5 claims). Each layer handles a different failure class and each is bounded, so total work is bounded.
**Detailed explanation:** A transient 503 should not consume a *step* attempt; a wrong patch should not be solved by re-calling the LLM with the same prompt. Separating layers keeps the cheap fix at the cheapest layer, and escalation after bounded effort.
**How this project uses it:** table in section 1; code in `llm/gemini.py`, `llm/structured.py`, `agents/loop.py`, `mcp_client.py`, `core/retry.py`, `worker.py`.
**Example/walkthrough:** Gemini 503 -> retried inside `complete()`; if it still fails -> `LLMTransientError` -> loop `error` -> step attempt 1 failed -> `step_retry` -> attempt 2 fails -> `escalated`.
**Follow-ups:**
- *Worst-case multiplication?* Up to 5 LLM tries per call, x loop steps, x 2 step attempts, x replans x 5 worker claims, which is why budgets exist. But budgets under-count across crashes and are not checked in planner/judge (see section 2).
- *Why isn't there backoff between step retries?* Not implemented; the LLM layer already backs off.
**Trade-offs and alternatives:** A generic retry decorator is simpler but cannot distinguish idempotent from non-idempotent calls.
**Common interview mistake:** Retrying everything everywhere.
**Related:** [06](06-llm-integration.md), [05](05-agents-and-orchestration.md).

## Q-REL-2. Why does the MCP client only retry some calls?
**Difficulty:** Advanced  **Topic:** Safe retries
**Short answer:** A timeout does not tell you whether the server executed the call. Retrying a read is harmless; retrying a write is only safe if the server dedupes by an idempotency key. So `_retryable` allows reads and keyed writes only.
**Detailed explanation:** Tool-level errors (`isError`) are deterministic answers and are not retried; transport errors and timeouts are ambiguous and are.
**How this project uses it:** `mcp_client.McpClient._retryable/call`; the key injection in `agents/tools.mcp_tooldefs`; servers enforce keys.
**Example/walkthrough:** `create_pull_request` times out; the client retries with the same key; the server returns the existing PR (`replayed=True`).
**Follow-ups:**
- *What if the tool has no key and it's a write?* One attempt; failure goes to the model as an error.
- *Is the key stable across retries within a call?* Yes, it is computed once from `(task, step, attempt, tool, args)` before the call.
**Trade-offs and alternatives:** At-most-once with no retry is safer but less resilient.
**Common interview mistake:** "We retry with exponential backoff" without mentioning idempotency.
**Related:** [15](15-mcp-servers-and-tool-layer.md), [07](07-state-durability-and-persistence.md).

## Q-REL-3. What does the escalation path look like to a user?
**Difficulty:** Beginner  **Topic:** Escalation
**Short answer:** The task ends with status `escalated` and an error message saying why and "needs human attention"; the report includes plan history, verification results and the last error. Approvals that expire also escalate with "nothing was published".
**Detailed explanation:** Escalation is not an exception; it is a status that routes to `finish`, so the report and events are produced the same way as success.
**How this project uses it:** `GraphNodes.dispatch` (step failure), `replan` (ladder end), `PublishNodes.await_decision` (expiry), `finish`, `build_report`.
**Example/walkthrough:** `test_repeated_failure_changes_strategy_then_escalates_to_a_human` asserts the status and message.
**Follow-ups:**
- *Can a human resume an escalated task?* There is no resume of a terminal task; `POST /tasks/{id}/retry` creates a new task with the same goal. (Verified in `api/app.py`.)
**Trade-offs and alternatives:** Pausing for guidance instead of ending would preserve context but adds a new state.
**Common interview mistake:** Treating escalated as failed.
**Related:** [04](04-end-to-end-workflows.md).

## Q-REL-4. How do budgets work, and what are their holes?
**Difficulty:** Advanced  **Topic:** Budgets
**Short answer:** Steps, tokens, dollars and wall-clock are tracked in `Budget`, shared by parallel agents in a node and checked at each agent-loop iteration; exhaustion stops the agent and escalates without retry. Holes: only the agent loop checks; wall-clock includes human waiting time; crash loses a node's spend; parallel overshoot; cost uses a local price table.
**Detailed explanation:** See section 2 gaps. Cost comes from `llm/pricing.estimate_cost` (an estimate, not provider billing).
**How this project uses it:** `core/state.Budget.exhausted`, `core/budget.BudgetTracker`, `agents/loop.py` (only caller of `exhausted`).
**Example/walkthrough:** Wall-clock scenario: approval waits 2 h, human rejects, replan, new coding step starts: first loop iteration returns `budget` ("wall-clock budget exhausted (1800s)") and the task escalates. (Derived from code; not a test.)
**Follow-ups:**
- *How would you fix the wall-clock issue?* Track *active* seconds (exclude `awaiting_approval` time) or reset `started_at` on resume.
- *Does the API let a user set an unbounded cost?* No: `TaskRequest.max_cost_usd` is validated `0 < x <= 50`; the default `Budget` is $5.
**Trade-offs and alternatives:** Hard preemptive limits (cancel in-flight call) vs cooperative checks.
**Common interview mistake:** Claiming a hard $ cap; it is a soft, per-iteration check.
**Related:** [06](06-llm-integration.md), [20](20-observability-and-telemetry.md).

## Q-REL-5. What happens when a specialist crashes or times out in the middle of parallel dispatch?
**Difficulty:** Intermediate  **Topic:** Partial failure
**Short answer:** `run_one` converts exceptions into an error result; the other specialist's result is kept; only the failed step goes back to `pending` for one retry. A2A timeouts do the same after trying other endpoints. If the whole worker dies mid-dispatch, the node (and both results) is redone.
**Detailed explanation:** Results are applied per step after `asyncio.gather`. `asyncio.gather` without `return_exceptions` would cancel others on first exception; the explicit try/except in `run_one` avoids that.
**How this project uses it:** `GraphNodes.dispatch/run_one`, `A2ASpecialist.run`.
**Example/walkthrough:** analytics errors, research finishes: `s2` done with artifacts; `s1` pending (attempt 1); next `dispatch` runs only `s1`.
**Follow-ups:**
- *Does the successful specialist's work survive a crash?* No, only after the node's checkpoint.
**Trade-offs and alternatives:** Per-step nodes would checkpoint each result at the cost of more graph machinery.
**Common interview mistake:** Forgetting `gather` cancels siblings on exception unless handled.
**Related:** [Q-STATE-3 in 07](07-state-durability-and-persistence.md).

## Q-REL-6. What does the system do when the human never answers?
**Difficulty:** Intermediate  **Topic:** Timeouts for humans
**Short answer:** Each approval has an expiry (default 24 h). `Worker.housekeeping` runs `approvals.expire_due()` every sweep; expired requests enqueue a resume with an `expired` payload; `await_decision` marks the publish step failed, sets `escalated` and says nothing was published. A decision after expiry gets HTTP 410.
**Detailed explanation:** The task is not holding a lease or a thread while paused, so waiting is free. The expiry sweep is part of the worker loop, so if **no worker is running** nothing expires until one starts (the `POST /approvals/expire` endpoint can also trigger it).
**How this project uses it:** `store/approvals.expire_due/decide`, `worker.housekeeping`, `hitl.expired_payload`.
**Example/walkthrough:** `test_sweeper_expires_overdue_approvals_and_resumes_their_tasks`.
**Follow-ups:**
- *Is `decide` also safe against expiry races?* The UPDATE requires `expires_at > now()`; otherwise it marks expired and queues the expired resume in the same transaction.
**Trade-offs and alternatives:** Reminders/escalation to another approver are not implemented.
**Common interview mistake:** Holding the worker blocked on a human.
**Related:** [04](04-end-to-end-workflows.md), [13](13-verification-and-risk-policy.md).

## Q-REL-7. Is there graceful shutdown?
**Difficulty:** Advanced  **Topic:** Operations
**Short answer:** No. The worker CLI installs no signal handler (its `stop` event is never set), and the embedded API worker is cancelled on shutdown on purpose. Recovery relies on lease expiry, so a deploy restarts in-flight nodes after up to ~30 s.
**Detailed explanation:** See section 5. This is consistent with a crash-only design but costs money and latency during deploys.
**How this project uses it:** `cli._worker`, `api/app.py` lifespan.
**Example/walkthrough:** `docker compose restart api` mid-run: container gets SIGTERM, process exits; after the lease expires the next worker (including the restarted one) claims and resumes.
**Follow-ups:**
- *What would a drain implementation do?* Set `stop`, finish or release the current claim, and set `status='queued', worker_id=NULL` immediately (so no lease wait).
**Trade-offs and alternatives:** Crash-only software (everything restartable) is a legitimate philosophy; drain is an optimisation.
**Common interview mistake:** Claiming graceful shutdown because the code mentions `stop`.
**Related:** [21](21-deployment-and-infrastructure.md), [07](07-state-durability-and-persistence.md).

## Q-REL-8. How do you know the reliability claims are true?
**Difficulty:** Advanced  **Topic:** Verification of claims
**Short answer:** Three evidence tiers: pure unit tests (`tests/core/test_retry.py`, `tests/graph/test_routing.py`), Postgres-backed store/e2e tests (`tests/store`, `tests/graph`) with scripted fake LLMs, and the opt-in chaos suite (`pytest -m chaos`) that SIGKILLs real worker processes. Not covered: live-LLM variance, A2A under chaos, cancel mid-run, DB outage, multi-host.
**Detailed explanation:** Default `addopts` excludes `live` and `chaos`, so CI runs the first two tiers; chaos requires Docker and minutes of runtime.
**How this project uses it:** `pyproject.toml` markers; `tests/chaos/test_chaos.py` invariants.
**Example/walkthrough:** `assert_exactly_once` checks PR count, branch list, commit count, approval and audit counts, and leftover containers.
**Follow-ups:**
- *Did you run the chaos suite?* I (the documentation author) read it but did not execute it; say "the suite exists and asserts X", not "I measured Y".
**Trade-offs and alternatives:** Property-based/stateful testing could explore more interleavings.
**Common interview mistake:** Citing test counts instead of invariants.
**Related:** [22](22-testing-strategy.md).

## Q-REL-9. What partial states can a failed or rejected publish leave behind?
**Difficulty:** Advanced  **Topic:** Side-effect residue
**Short answer:** A pushed `aeos/<task>-v<n>` branch with the commit stays on the remote after rejection, expiry, cancel or a later failure; a pending approval is closed on cancel; the local PR row/GitHub PR exists only after approval. Nothing deletes branches.
**Detailed explanation:** `publish_prepare` makes the branch and commit before human approval because the policy classifies them reversible; the irreversible step (PR) is gated. Rejections replan to a new version-suffixed branch.
**How this project uses it:** `graph/publish.py::prepare`; policy rules `reversible-branch-writes`, `never-merge` (denies `delete_branch`).
**Example/walkthrough:** Two rejections leave `aeos/t-x-v1` and `aeos/t-x-v2` on the repo.
**Follow-ups:**
- *Is that a security concern?* Branches are namespaced and cannot touch default branches; they are clutter, not a risk. Pushing happens through the github MCP server with the configured token.
**Trade-offs and alternatives:** Create the branch only after approval, at the cost of showing the human a diff rather than a real branch.
**Common interview mistake:** Not mentioning residue at all.
**Related:** [16](16-a2a-and-github-integration.md), [13](13-verification-and-risk-policy.md).

## Q-REL-10. If you had one day, which reliability gaps would you close?
**Difficulty:** Expert  **Topic:** Prioritisation
**Short answer:** (1) Validate `approval_id` on resume (safety boundary); (2) exclude human-wait time from the wall-clock budget (breaks real use); (3) SIGTERM drain with explicit lease release; (4) budget checks around planner/judge/synthesis calls; (5) reset or redefine `attempts`. Each is small and local.
**Detailed explanation:** Ordered by impact: (1) touches the human-approval guarantee in a rare race, (2) is a likely day-one functional bug for slow approvals, (3) improves deploys, (4)/(5) harden limits.
**How this project uses it:** `PublishNodes.await_decision`, `Budget`, `cli._worker`, `agents/supervisor.py`, `store/tasks.claim`.
**Example/walkthrough:** For (2), add `waiting_seconds` to `Budget`, accumulate in `await_decision` using `approval.created_at/decided_at`, subtract in `exhausted()`.
**Follow-ups:**
- *How would you test (1)?* A store+graph test: stale rejected payload with a later interrupt must not be applied.
- *What would you not do?* Replace LangGraph with Temporal; the current guarantees are met at this scale (D-033).
**Trade-offs and alternatives:** Each fix adds code for rare conditions; judge by impact.
**Common interview mistake:** Proposing a rewrite instead of targeted fixes.
**Related:** [24](24-technical-challenges-and-history.md), [07](07-state-durability-and-persistence.md).

## Q-REL-11. A task is stuck in `running` for minutes. How do you diagnose it?
**Difficulty:** Intermediate  **Topic:** Operations
**Short answer:** Check `aeos.tasks` (`worker_id`, `lease_expires_at`, `heartbeat_at`, `attempts`, `last_error`). If the lease is expired and a worker is running, it will be reclaimed within one poll. If heartbeats are fresh, look at the timeline/trace: it is probably inside a long agent loop or sandbox command.
**Detailed explanation:** `GET /tasks/{id}` shows `queue_status`, `attempts`, `status_line`; `GET /tasks/{id}/events` shows the last events; Langfuse shows the node span in progress.
**How this project uses it:** `api/app.py::task_view`, `store/tasks.TaskRow`.
**Example/walkthrough:** `attempts=3` with `last_error` set means it was requeued by `release_with_error` twice.
**Follow-ups:**
- *What if events stopped but heartbeat continues?* The graph may be blocked on an LLM call with no timeout (see section 2).
**Trade-offs and alternatives:** Add a "stuck" detector (no event for N minutes) (Recommendation).
**Common interview mistake:** Restarting before reading `attempts` (the crash-loop guard counts claims).
**Related:** [20](20-observability-and-telemetry.md), [18](18-api-and-client.md).
