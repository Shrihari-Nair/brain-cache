# 32 - Rapid revision (tag: RAPID)

## What this file covers

One-page cheat sheets for the last day before an interview: a 60-second pitch, a 5-minute pitch, key numbers that were re-checked against the repo (each with its source file), a component table, a glossary, the top 25 facts, the top 15 things not to overclaim, and a last-hour checklist. Everything here points to a deeper file; nothing here is new analysis. Deeper material: [01-project-overview.md](01-project-overview.md), [28-senior-engineer-followups.md](28-senior-engineer-followups.md), [31-mock-interviews.md](31-mock-interviews.md). Tag RAPID has no Q&A section by design; the pitches and "say it out loud" lists are the drill.

---

## 1. The 60-second pitch (say it aloud, about 150 words)

"AEOS, the Autonomous Engineering OS, takes an engineering request, like 'why did checkout latency regress' or a code change on a real repo, and ends with a pull request that a human has approved. A supervisor makes a plan; specialist agents investigate and write a patch in a Docker sandbox with no network; then an independent verifier re-applies the patch to a fresh checkout and re-runs the tests and a benchmark itself. A human approves the exact pull request, and only then is a real GitHub PR opened. It never merges. All progress is in Postgres, so if I kill -9 the worker it resumes, and a chaos test shows one PR, one branch, one commit even when killed at about two dozen different points. On 30 synthetic tasks one live run passed 26, which I treat as a narrow, uncontrolled number. The point of the project is the control plane around the models."

## 2. The 5-minute pitch (structure with timings)

| Minute | Say | Pointer |
|---|---|---|
| 0:00 | Problem: demo agents say "done"; the hard part is trust, crashes, injection and measurement | [01](01-project-overview.md) |
| 0:45 | Architecture: supervisor -> specialists (in-process or A2A) -> verifier -> approval -> PR; LangGraph with 10 nodes and a pure `route` function | [03](03-system-architecture.md), [05](05-agents-and-orchestration.md) |
| 1:45 | Verification: fresh sandboxes, gates (tests, benchmark, diff size, forbidden paths, secrets), `regression_reproduced`; judge can only tighten | [13](13-verification-and-risk-policy.md) |
| 2:30 | Durability: lease queue, checkpoints, idempotency keys, compare-and-set approvals, chaos tests | [07](07-state-durability-and-persistence.md) |
| 3:30 | Safety: YAML risk policy (deny wins), `aeos/*` only, no merge, sandbox flags, SQL guard, taint labels | [12](12-sandbox-and-execution-safety.md), [14](14-security-and-prompt-injection.md) |
| 4:15 | Evals: graders recompute outcomes; 26/30; not controlled; judge validation small | [23](23-evaluation-framework.md) |
| 4:45 | Limits and next steps: auth default off, docker socket, budget holes, verifier holes | [28](28-senior-engineer-followups.md) |

## 3. Key numbers (verified; source in the third column)

| Number | Meaning | Source (repo-relative) |
|---|---|---|
| about 12.5k lines | Python in `src/aeos` (counted with `wc -l` over all `.py`: 12,509) | `src/aeos/**` |
| 476 | test functions found by grep for `def test_` (parametrised cases not expanded) | `tests/**` |
| 10 | LangGraph nodes: plan, dispatch, synthesize, verify, replan, publish_prepare, await_decision, publish_pr, finish, reflect | `src/aeos/graph/build.py` |
| 120 | LangGraph recursion limit | `graph/build.py::_config` |
| 200 / 1,500,000 / 5.0 / 1800 | default budget: steps / tokens / estimated USD / wall seconds | `src/aeos/core/state.py::Budget` |
| 25 / 3 | per-agent-loop max steps / max malformed (no tool call) responses | `src/aeos/agents/loop.py::LoopLimits` |
| 3 | verification failures before escalation | `src/aeos/core/retry.py` (`max_verification_failures`) |
| 30 s / lease/3 | worker lease / heartbeat period | `src/aeos/worker.py` |
| 5 | claims before crash-loop escalation | `src/aeos/store/tasks.py::claim` (`max_attempts`) |
| 86,400 s | approval TTL | `src/aeos/core/config.py` (`approval_ttl_s`) |
| 5 files / 300 lines / 10 percent | verifier diff limits and p95 tolerance | `src/aeos/agents/verifier.py::GateConfig` |
| 12,000 chars | patch window the judge sees | `src/aeos/agents/judge.py` |
| 23 | chaos kill points: 8 nodes x before/after = 16, plus 7 tool points | `tests/chaos/test_chaos.py` (`POINTS`) |
| 8 commits | git history, snapshots not a dev log | `git log` |
| 30 tasks, 6 templates | eval dataset | `evals/datasets/tasks.yaml`; `evals/results/20261012T095434Z_gemini-baseline-oct12.md` |
| 0.867 +/- 0.062 | task success, 12 Oct live run, n=30, 1 trial each | `evals/results/20261012T095434Z_gemini-baseline-oct12.md` |
| 0.967 | verified fix delivered (root-cause commit not required) | same |
| 0.883 | root cause correct | same |
| 0.500 (n=2) | recovery after failed attempt (meaningless sample) | same |
| 0.065 USD / 25.6 calls / 22.4 steps / 116 s | per-task averages in that run | same |
| 0.500 (15/30) | 3 Oct live run success; includes timeouts and judge 404 | `evals/results/20261003T024031Z_live-gemini-full.md` |
| 12 labelled reports | judge validation set; README reports Spearman 0.80, no stored output | `evals/judge_validation.yaml`, `README.md` |

Note: the knowledge-base files may say 22 in places; the file as read gives 16 + 7 = 23 parametrised points. Recount `POINTS` before quoting an exact figure.

## 4. Component table

| Component | Where | One-line role | Caveat to know |
|---|---|---|---|
| Graph / routing | `graph/build.py`, `routing.py`, `nodes.py`, `publish.py` | State machine; `route` is pure | `_crashable` wraps nodes for kill tests |
| Supervisor | `agents/supervisor.py` | LLM plan + code validation + appended verify/publish | plan quality is not code-checked |
| Specialists | `agents/specialists.py` | analytics, research, coding loops | A2A optional |
| Agent loop | `agents/loop.py` | tool loop with finish tool, limits | budget only checked here |
| Tool runtime | `agents/tools.py` | risk gate, idempotency key, origin labels | `</tool_result>` not escaped |
| Risk policy | `core/risk.py`, `policy/risk_policy.yaml` | deterministic allow/approve/deny | taint rule ordered after allow rules |
| Verifier | `agents/verifier.py`, `verifier_change.py`, `suite.py` | recompute gates | lint = compileall |
| Judge | `agents/judge.py` | advisory strict-only | sees 12k chars |
| Sandbox | `sandbox/*` | Docker, no network, read-only | socket mounted; npm has network |
| MCP servers | `mcp_servers/{github,database,observability,web}` | tools with declared risk | `mcp>=1.2` open pin |
| A2A | `a2a/*` | remote specialists | auth optional, in-memory store |
| Queue | `store/tasks.py`, `worker.py` | leases, fencing, attempts | attempts also counts resumes |
| Checkpoint | `graph/checkpoint.py` | Postgres saver | no schema versioning |
| Approvals | `store/approvals.py`, `hitl.py` | CAS decision, TTL, audit | hash and args in same row |
| Memory | `memory/*` | pgvector episodes/facts, write gate | no demonstrated benefit |
| LLM layer | `llm/*` | provider interface, Gemini, fake, pricing | `strong_llm` not separately wired |
| Telemetry | `telemetry/*` | OTel traces, Langfuse grouping | logs lack ids; metrics debug only |
| API / client | `api/*`, `client/*` | FastAPI, CLI | auth off by default |
| Dashboard | `web/` | Next.js UI | |
| Evals | `evals/*` | graders, metrics, compare | not in CI |
| Infra | `docker-compose.yml` | Postgres, services, Redis | Redis unused |

## 5. Glossary

| Term | Meaning (in AEOS) |
|---|---|
| Checkpoint | LangGraph state snapshot saved to Postgres after each node |
| Thread id | LangGraph's key for a run; equals the task id |
| `interrupt()` | Pause the graph and surface a payload; resume with `Command(resume=...)` |
| Lease | Time-limited claim on a task row; renewed by heartbeat |
| Fencing | Writes require current `worker_id`, so a stale worker cannot overwrite |
| `SKIP LOCKED` | Postgres clause so concurrent claimers skip rows another transaction holds |
| Idempotency key | Deterministic string making a write safe to replay |
| Compare-and-set | `UPDATE ... WHERE status='pending'` so a decision applies once |
| Risk level | `read`, `write_reversible`, `write_irreversible`, declared by the tool's server |
| Origin / taint | `trusted` vs `untrusted` result label; `untrusted_only` once an agent read untrusted data |
| Grant | Human approval bound to tool + argument hash |
| Gate | Deterministic verifier check producing true/false |
| Judge | LLM reviewer, advisory, can only tighten |
| MCP | Protocol for agent-to-tool servers |
| A2A | Protocol for agent-to-agent delegation |
| Agent card | A2A discovery document, public here |
| Escalated | Terminal status meaning a human is needed |
| Replan | New plan version after failed verification or rejection |
| Reaper | Removes orphan sandboxes (worker housekeeping; standalone CLI unscheduled) |
| HashEmbedder | Offline feature-hash embedder; vs `GeminiEmbedder` |
| pgvector / HNSW | Vector column type / approximate nearest-neighbour index |
| Spearman | Rank correlation, used for judge-vs-human agreement |
| Stderr | Standard error of the mean, over tasks (population std, small-n understated) |
| `code_version` | Five hashes (code, prompts, policy, dataset, graders) in each result file |
| Chaos test | kill -9 at named points, then assert exactly-once effects |
| Red herring | Decoy commit in `demo_target` that must be ruled out |

## 6. Top 25 facts

1. The system opens a PR and never merges; `never-merge` is a deny rule.
2. Agents may only write to `aeos/*` branches (deny rule `branch-namespace`, also validated in the GitHub server and `remote.py`).
3. `route(state)` is a pure function; the LLM never picks the next node.
4. The verifier step and the publish step are appended by code (`finalize_plan`).
5. The patch artifact comes from the sandbox snapshot, not model text.
6. `_check_research` replaces the model's baseline with `<suspect>~1` and rejects non-commit SHAs.
7. The verifier re-applies the patch in fresh sandboxes and reruns tests and benchmark itself.
8. `regression_reproduced` exists because a wrong baseline made the gate pass trivially.
9. The judge can only make a verdict stricter (`apply_judgement`).
10. Policy: any deny wins; else first matching non-deny; else default `require_approval`.
11. An approval grant is bound to tool + sha256 of canonical arguments (minus idempotency key).
12. Approval decisions are compare-and-set; the resume payload is stored in the same transaction.
13. Publish is three nodes so resume never repeats branch/commit work.
14. Idempotency key = task | step | attempt | tool | canonical args.
15. Crash recovery re-runs the interrupted node; effects are protected by keys, LLM work and spend are repeated.
16. Leases: 30 s, heartbeat each lease/3, `FOR UPDATE SKIP LOCKED`, fenced by `worker_id`.
17. Five claims without success escalate a task (crash-loop guard); resumes also count.
18. The sandbox has `network_mode="none"`, read-only root, `cap_drop=ALL`, memory and pid limits.
19. SQL tool: AST-based single-SELECT guard plus a read-only DB role.
20. Memory uses pgvector with HNSW cosine; writes pass a deterministic gate; no benefit was shown.
21. Tracing: OTel, trace context stored in state so resumed runs stay in one trace; Langfuse via OTLP.
22. Evals: graders recompute outcomes; rubric score excluded from success.
23. 12 Oct live run: 26/30 success, 29/30 verified fix, 0.065 USD per task.
24. First live run lost its results because comparison ran before the save; fixed in `2fab6fc`.
25. Git history is 8 snapshot commits; do not narrate it as a development diary.

## 7. Top 15 things NOT to overclaim

| # | Do not say | Say instead |
|---|---|---|
| 1 | "The verifier guarantees correctness" | "It re-checks a defined spec; a human approves" |
| 2 | "Exactly-once" (unqualified) | "At-least-once execution with idempotent effects; chaos-tested with a fake LLM on the local remote" |
| 3 | "87% of bugs fixed" | "26/30 on one synthetic family, one trial each, stderr 0.06" |
| 4 | "Improved from 50% to 87% because of X" | "Code, prompts and graders all changed; not controlled" |
| 5 | "Judge validated at Spearman 0.80" | "12 items, at threshold, no stored output" |
| 6 | "Prompt injection is prevented" | "Blast radius is bounded; the model can still be fooled" |
| 7 | "Sandbox is secure" | "Good boundary against accidents; shared kernel; socket mounted" |
| 8 | "Approval is tamper-proof" | "Bound to a hash stored in the same row" |
| 9 | "Hard cost cap" | "Soft, loop-level, estimated; planner/judge not gated" |
| 10 | "Production-ready" | "Local-stack prototype; auth off by default" |
| 11 | "Memory improves results" | "Mechanism built and gated; no benefit measured" |
| 12 | "Strong model used for the judge" | "Wiring exists; production runtime reuses the same model" |
| 13 | "Dashboard/observability complete" | "Traces good; log ids null, health check constant" |
| 14 | "Redis is used for X" | "Provisioned, unused" |
| 15 | "Lint gate" | "Syntax compile only" |

## 8. Last-hour checklist

- [ ] Say the 60-second pitch aloud twice; time it.
- [ ] Say the end-to-end flow (12 steps in Q-MOCK-4) without notes.
- [ ] Recite: verifier gates, policy order, idempotency key, lease numbers.
- [ ] Recite the three honest limitations you volunteer first: auth default off; verifier holes (conftest, compileall); budget holes.
- [ ] Recall the 12 Oct numbers and their caveats (denominator, family, one trial, uncontrolled).
- [ ] Prepare one documented story (live-eval baseline bug or lost-results fix) with root cause, fix, test.
- [ ] Prepare one thing you would cut (A2A, Redis, memory).
- [ ] Re-read section 7 above.
- [ ] Open `core/risk.py::decide`, `agents/verifier.py::_run_gates`, `graph/publish.py` once, so file names are fresh.
- [ ] Do not memorise numbers you have not seen in section 3; if unsure, say "roughly" and name the file.
- [ ] If asked about incidents: say there are no production incidents; offer the documented development findings instead.
