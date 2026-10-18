# AEOS (Autonomous Engineering OS): Interview Preparation Guide

Source repo: `autonomous-engineering-os/` (Python, LangGraph, Postgres, Docker sandbox, MCP, A2A, FastAPI, Next.js).
Everything here was written from reading the actual code, tests and git history. Each claim is labelled **Verified**, **Inferred**, **Potential failure scenario** or **Recommendation**. Nothing in this guide is a measured benchmark unless it cites a stored result file.

## The project in one paragraph

AEOS takes an engineering request in plain English ("update the `ws` dependency and raise a PR", or "find why checkout latency regressed"). A supervisor plans the work. Research and analytics agents investigate through MCP tools. A coding agent changes code inside a network-less Docker sandbox. An independent verifier re-applies the patch to a fresh checkout and runs deterministic gates, with an LLM judge that can only make the verdict stricter. A human approves, and only then does AEOS push an `aeos/*` branch and open a real GitHub PR. State is checkpointed to Postgres so a killed worker resumes without repeating side effects. Slogan to defend: **agents decide, code enforces.**

## Recommended learning path

| Stage | Goal | Read |
|---|---|---|
| 1 | What the project does | [01 overview](01-project-overview.md), [02 repo tour](02-repository-tour.md) |
| 2 | Architecture and end-to-end flows | [03 architecture](03-system-architecture.md), [04 workflows](04-end-to-end-workflows.md), `diagrams/` |
| 3 | Underlying concepts | [05 agents](05-agents-and-orchestration.md), [06 LLM layer](06-llm-integration.md), [07 state and durability](07-state-durability-and-persistence.md), [08 memory](08-memory-system.md), [15 MCP](15-mcp-servers-and-tool-layer.md), [16 A2A and GitHub](16-a2a-and-github-integration.md) |
| 4 | Decisions and trade-offs | [25 decisions](25-design-decisions-and-tradeoffs.md), [12 sandbox](12-sandbox-and-execution-safety.md), [13 verification](13-verification-and-risk-policy.md), [14 security](14-security-and-prompt-injection.md) |
| 5 | Debugging, testing, production | [10 reliability](10-error-handling-and-reliability.md), [22 testing](22-testing-strategy.md), [23 evals](23-evaluation-framework.md), [20 observability](20-observability-and-telemetry.md), [21 deployment](21-deployment-and-infrastructure.md), [27 incidents](27-debugging-and-incident-scenarios.md), [18 API](18-api-and-client.md), [19 dashboard](19-web-dashboard.md) |
| 6 | Senior-level follow-ups | [28 senior follow-ups](28-senior-engineer-followups.md), [26 scalability](26-scalability-and-system-design.md), [29 walkthroughs](29-code-walkthroughs.md), [30 limitations](30-limitations-and-improvements.md), [24 history](24-technical-challenges-and-history.md) |
| 7 | Mocks and revision | [31 mock interviews](31-mock-interviews.md), [32 rapid revision](32-rapid-revision.md) |

**Prerequisites to skim first:** async Python basics, what a state machine / DAG is, Docker (namespaces, cgroups, capabilities), Postgres transactions and row locking, JSON-RPC, function calling (tool use) with LLMs, embeddings and cosine similarity, OpenTelemetry spans.

## Document index

| File | Covers |
|---|---|
| [01-project-overview](01-project-overview.md) | Purpose, users, task types, how to pitch it |
| [02-repository-tour](02-repository-tour.md) | Annotated directory map, module table, read-first list |
| [03-system-architecture](03-system-architecture.md) | Components, boundaries, runtime topology |
| [04-end-to-end-workflows](04-end-to-end-workflows.md) | Change request, regression, replan, approval, cancel, crash-resume traces |
| [05-agents-and-orchestration](05-agents-and-orchestration.md) | Supervisor, specialists, tool-calling loop, replan |
| [06-llm-integration](06-llm-integration.md) | LLM abstraction, Gemini, structured output, cost, FakeLLM |
| [07-state-durability-and-persistence](07-state-durability-and-persistence.md) | TaskState, checkpointer, leases, idempotency, event log |
| [08-memory-system](08-memory-system.md) | Embeddings, store, write gate, recall, poisoning risks |
| [10-error-handling-and-reliability](10-error-handling-and-reliability.md) | Retries, budgets, failure table, what is and is not handled |
| [12-sandbox-and-execution-safety](12-sandbox-and-execution-safety.md) | Docker hardening, command policy, reaper, npm container |
| [13-verification-and-risk-policy](13-verification-and-risk-policy.md) | Verifier gates, judge, risk policy YAML, approval binding |
| [14-security-and-prompt-injection](14-security-and-prompt-injection.md) | Threat model, implemented-vs-missing table |
| [15-mcp-servers-and-tool-layer](15-mcp-servers-and-tool-layer.md) | MCP protocol, four servers, 23 tools, SQL guard |
| [16-a2a-and-github-integration](16-a2a-and-github-integration.md) | A2A vs MCP, publish flow, idempotency, demo_target |
| [18-api-and-client](18-api-and-client.md) | FastAPI, auth, SSE, CLI and terminal client |
| [19-web-dashboard](19-web-dashboard.md) | Next.js app, pipeline, trace waterfall, approvals |
| [20-observability-and-telemetry](20-observability-and-telemetry.md) | Spans, OTel/Langfuse, logging, cost stats |
| [21-deployment-and-infrastructure](21-deployment-and-infrastructure.md) | Compose, Docker, CI, tooling, production gaps |
| [22-testing-strategy](22-testing-strategy.md) | Test suites, chaos tests, untested areas |
| [23-evaluation-framework](23-evaluation-framework.md) | Datasets, graders, judge validation, stored results |
| [24-technical-challenges-and-history](24-technical-challenges-and-history.md) | Git timeline and honest challenge narration |
| [25-design-decisions-and-tradeoffs](25-design-decisions-and-tradeoffs.md) | 19 decisions with alternatives |
| [26-scalability-and-system-design](26-scalability-and-system-design.md) | 10x/100x analysis, bottlenecks, multi-instance |
| [27-debugging-and-incident-scenarios](27-debugging-and-incident-scenarios.md) | 13 incident scenarios plus a debugging toolkit |
| [28-senior-engineer-followups](28-senior-engineer-followups.md) | Skeptical questions and what not to overclaim |
| [29-code-walkthroughs](29-code-walkthroughs.md) | 8 key execution paths |
| [30-limitations-and-improvements](30-limitations-and-improvements.md) | 32 verified limitations, 22 proposals |
| [31-mock-interviews](31-mock-interviews.md) | 7 rounds with expected answers |
| [32-rapid-revision](32-rapid-revision.md) | Pitches, verified numbers, glossary, last-hour checklist |
| `diagrams/` | `system-architecture.mmd`, `primary-workflow.mmd`, `state-machine.mmd` (more Mermaid diagrams are inline in the documents) |

(File numbers 09, 11 and 17 are intentionally unused.)

## The five ideas to be able to explain without notes

1. **Agents decide, code enforces.** Risk policy, sandbox limits, verifier gates, SQL guard, idempotency and approval binding are code, not prompts ([13](13-verification-and-risk-policy.md), [14](14-security-and-prompt-injection.md)).
2. **Independent verification.** Patch re-applied to a fresh checkout; the LLM judge can only tighten ([13](13-verification-and-risk-policy.md)).
3. **Durability.** Postgres checkpoints, worker leases, deterministic idempotency keys, at-least-once execution with idempotent effects ([07](07-state-durability-and-persistence.md)).
4. **Human-in-the-loop bound to what was seen.** Approval stores a hash of the exact PR arguments ([13](13-verification-and-risk-policy.md)).
5. **MCP for tools, A2A for agents.** Different problems, both optional layers ([15](15-mcp-servers-and-tool-layer.md), [16](16-a2a-and-github-integration.md)).

## Important limitations and open questions (all Verified in code, see [30](30-limitations-and-improvements.md))

- Git history is 8 snapshot commits, not a development log. Do not narrate blockers the history cannot support ([24](24-technical-challenges-and-history.md)).
- Live eval 15/30 (3 Oct) vs 26/30 (12 Oct) is not a controlled comparison; the README judge-validation figure has no stored output ([23](23-evaluation-framework.md)).
- Redis is provisioned but unused; the "strong model for planner" wiring is unused in production.
- Budget is checked only in the agent loop; wall-clock includes approval wait; `attempts` counts approval resumes.
- No graceful shutdown; `cancelled` status is never set in the graph state; `aeos/*` branches are never cleaned.
- Verifier: "lint" gate is only `compileall`; `GateConfig.secret_values` is never populated; judge sees `patch[:12000]`.
- Docker socket mounted into app services; npm container has network; A2A auth is optional.
- Auth is off by default; CI has no Postgres service so DB tests likely skip; dashboard has no tests.
- Logging: worker/serve run at ERROR, `run_context` never called so log ids are null; metrics go only to the debug exporter.

## Corrections made during review

- Sandbox reaper: `Worker.housekeeping` (`src/aeos/worker.py`) does call `sandbox.reap`. Only the standalone `python -m aeos.sandbox.reaper` CLI is unscheduled. Files 12 and 14 were corrected.
- The judge is **tighten-only**, not "advisory-only": a judge "no" can fail a patch that passed every gate.
- Chaos matrix: 23 parametrised kill points + 3 random-kill seeds + 1 crash-while-paused = 27 tests (uses the fake LLM and the local bare remote).

## Preparation checklist

- [ ] Stage 1: can give the 60-second and 5-minute pitch ([32](32-rapid-revision.md))
- [ ] Stage 2: can draw the architecture and trace a change request end to end
- [ ] Stage 3: can explain LangGraph checkpointing, MCP, A2A, sandboxing, embeddings from first principles
- [ ] Stage 4: can defend 5 decisions and name an alternative for each
- [ ] Stage 5: can walk through 3 incident scenarios using real tables and endpoints
- [ ] Stage 6: can answer the senior follow-ups without overclaiming
- [ ] Stage 7: completed all mock rounds and the last-hour checklist
