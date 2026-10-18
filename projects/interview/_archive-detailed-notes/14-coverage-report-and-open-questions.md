# 14 — Coverage Report and Open Questions

## 14.1 What was produced

| Doc | Purpose |
|---|---|
| [00](00-learning-roadmap.md) | Roadmap, 15-minute overview, evidence labels, index |
| [01](01-fundamentals.md) | Prerequisite concepts |
| [02](02-technology-deep-dives.md) + 02a–02e | Technology deep dives (backend/data, AI/LLM, voice/LiveKit, frontend, infra/tooling) |
| [03](03-system-architecture.md) | Architecture and diagrams |
| [04](04-codebase-walkthrough.md) | File/symbol walkthrough |
| [05](05-end-to-end-workflows.md) | 11 end-to-end flows incl. failure paths |
| [06](06-core-concepts-and-implementation.md) | Core mechanisms in detail |
| [07](07-engineering-challenges.md) | Confirmed issues / observed / limitations / hypotheses / improvements |
| [08](08-performance-reliability-and-scaling.md) | Performance, reliability, scaling (measured vs reasoned) |
| [09](09-testing-evaluation-and-observability.md) | Tests, evals, tracing, commands |
| [10](10-design-decisions-and-tradeoffs.md) | Why it is designed this way |
| [11](11-interview-preparation.md) | Six-level interview prep |
| [12](12-practical-exercises.md) | Three-level exercises |
| [13](13-glossary-and-reference.md) | Glossary, keys, API, commands |

No application code, configuration, or `.gitignore` entry was changed for this work; files were written only under `docs/codebase-learning/`.

## 14.2 What I ran (and what I did not)

| Action | Outcome |
|---|---|
| `backend/.venv/bin/python -m pytest backend/tests -q` | 53 passed, 1 skipped, 17 warnings |
| `agent/agent.py --help` | prints CLI + a `DeprecationWarning` for `livekit.plugins.turn_detector` |
| Local analysis scripts (AST symbol index, import graph, git log/show, grep for absent features, counts over `evals/results/*.json`, a random-draw simulation) | informed [Measured] labels |
| Documentation cross-check script | 498 path/symbol/line references and all inter-doc links checked; remaining "problems" are library files outside the repo, `.github/workflows/*` paths (outside `mockmind/`), and a symbol-window false positive for `entrypoint` — no incorrect references found among them |
| **Not run:** evals (paid Gemini), frontend typecheck/build, Docker, live voice sessions, load tests | stated wherever it matters |

Network: none used. Secrets: `.env` values were never read into the docs (names only).

## 14.3 Coverage matrix (post-audit)

Status rule: **covered** = enough detail, with a worked example and failure paths, to explain and defend in an interview; **partial** = explained but thin, unrendered, unmeasured, or reading-derived; **not covered** = absent. Status reflects the audit agents' re-verification against source plus my own spot checks; it is not a claim of perfection.

| Component / concept | Main source files | Explained in | Status | Why / what is missing |
|---|---|---|---|---|
| FastAPI app, routers, CORS, lifespan | `backend/main.py`, `routers/*` | 02a, 13 | covered | error contracts and form params included |
| Preparation pipeline + SSE | `routers/prepare.py` | 02a, 05, 06 | covered | Starlette upload-spooling claim is Inference |
| Resume/JD analysis, question generation, level/coding slots | `backend/agents/*` | 02b, 06 | covered | |
| Pydantic models | `backend/models/*` | 02a, 06 | covered | |
| Redis keys/TTL/persistence | `redis_client.py`, `docker-compose.yml` | 02a, 03, 13 | covered | persistence measured on the local container only |
| Report pipeline + status machine | `report_pipeline.py`, `report_store.py` | 06, 05, 07 | covered | live success rate not measured |
| Candidate memory | `memory_*.py` | 06, 07 | covered | no concurrency test |
| LLM prompts, structured output, thinking budget | `agents/*`, `report_generator.py` | 02b, 06, 07 | covered | experiments not reproducible offline |
| Agent: interviewer, tools, state, follow-up cap | `agent/*` | 02b, 06, 05 | covered | |
| Voice pipeline (VAD/STT/turn/TTS), LiveKit | `agent/agent.py`, library source | 02c, 08 | covered | per-stage latency not measured; library claims checked against installed 1.8.4 |
| Data channels, coding flow, Piston | `agent.py`, `code_*` | 02c, 05, 06 | partial | first-run Piston runtime install vs 30 s timeout untested |
| Frontend (pages, hooks, components) | `frontend/*` | 02d, 05 | partial | light audit edits only; no browser run |
| Architecture diagrams | | 03, 00 | partial | Mermaid reviewed by eye, **never rendered** (ER diagram especially) |
| End-to-end workflows + failure paths | | 05 | partial | flows 5.1–5.3, 5.11 less deep; some timelines are Inference |
| Engineering challenges | | 07 | covered | evidence classes separated; hashes/numbers recounted |
| Performance/scaling | | 08 | partial | reasoning plus a few local measurements; no load tests (stated) |
| Testing/evals/CI | `tests/*`, `evals/*`, workflows | 09, 06, 02e | covered | eval internals in 06 §6.12 not deepened; judge uncalibrated |
| Observability/tracing | `tracing.py`, `agent.py` | 02e, 09 | partial | Langfuse UI content unverified |
| Docker/Compose/config/env | compose, Dockerfiles, `.env.example` | 02e, 13 | covered | compose never run in this audit |
| Design decisions | | 10 | covered | many rationales honestly "not recorded" |
| Interview bank | | 11 | partial | 86 questions; project-specific only, general screens and company prep not covered; behavioural answers need your real input |
| Exercises | | 12 | covered | offline ones run with real output; live-API ones marked |
| Glossary/reference | | 13 | partial | not every term from 02b–02d |

## 14.4 Concept inventory

Mapped (concept → where): async I/O (02a), ASGI/FastAPI (02a), Pydantic (02a), Redis TTL store (02a, 13), SSE (02a, 05), multipart/PDF parsing (02a), retry/backoff (06, 07), LLM structured output (02b), thinking budget (02b, 07), prompt injection (02b, 07), workflow-vs-agent (02b), tool calling (02b, 06), session state machine (06), follow-up policy (06, 07), WebRTC/LiveKit rooms (02c), explicit dispatch (02c, 06), VAD/STT/endpointing/TTS (02c), data channels (02c, 05), Monaco/code submission (02d, 05), Piston sandbox (02e, 06), JWT grants (06), report state machine (06, 07), candidate memory (06, 07), eval personas/checks/judge (06, 09), tracing (02e, 09), Docker/Compose/CI (02e).
**Searched for and found absent** (so documented as *not used*): RAG/embeddings/vector DB, multi-agent orchestration, authentication, message queue, SQL database, WebSockets (application-level), frontend tests. See negative-search table in [02b](02b-ai-and-llm-deep-dive.md).

## 14.5 Known inaccuracies in the repo's own documents
- README: "guarantee" of difficulty mix — the code draws randomly.
- `MOCK_INTERVIEW_PLATFORM.md`: historical; Deepgram vs Soniox, `tools.py` merged, question counts differ.
- `evals/REPORT.md`: refers to an "earlier audit" not present in the repo; describes the follow-up counter as unused at the time (later wired).
- A separate personal `study-notes/` folder (git-ignored) predates this set and may still say Piston is called by the "backend/agent"; this `docs/` set supersedes it (agent only).

## 14.6 Open questions (I could not answer from the repository)
1. Who diagnosed/fixed each historical issue, and was there an "earlier audit" with other findings?
2. Why was Deepgram replaced by Soniox, and why Cartesia `sonic-3` with that voice?
3. Actual live latency per turn, report success rate in real calls, silent-turn frequency in voice (vs text harness).
4. Does the report generator run to completion inside the job-shutdown window in practice (D1)? Needs a live test.
5. What happens with two tabs on one session id (D4)?
6. Has `evals.yml` ever been run on GitHub, and with what results (only local, git-ignored results were available)?
7. Root cause of the empty "silent" turn after a tool call (B1) and of interviewer role confusion (B2).
8. How the thinking-budget experiments were run (13/13 vs "6 trials" wording).
9. Production deployment target (none in repo); expected traffic.
10. Whether the code drift noted in [07](07-engineering-challenges.md) C12 (TS types vs Pydantic models, the unused `useInterviewSession` hook) is intended.

## 14.7 How to keep this documentation honest
- Re-run `pytest` and update the counts in 09/13 when tests change.
- Re-run a line-reference check after code edits (line numbers are as of this checkout; symbol names are the durable anchor).
- Treat any [Hypothesis]/[Inference]/[Not measured] item as unconfirmed until tested.

## 14.8 Audit report (second pass)

**Corrections made (verified):**
- Wrong line references fixed (`usePreparation.ts`, `report_generator.py`, `entrypoint`); an invalid Mermaid ER diagram rewritten.
- Redis persistence corrected everywhere: the compose Redis uses the image-default RDB snapshots, AOF is off, and `/data` is an anonymous volume, so recent writes can be lost; it is not "no persistence".
- `session_state` is read by `retry_report`; I had said it was not consumed.
- `NEXT_PUBLIC_LIVEKIT_URL` is not read by frontend code (the URL comes from the token response).
- Evals are a manual workflow, not automatic CI; a `DELETE /api/candidate-memory` endpoint exists.
- Commit messages for `aa3e481`, `f2243ad`, `568bffc` don't fully match their contents; the follow-up hook first appears in `568bffc`.
- Spec/code differences recorded without invented rationale (`create_dispatch`, `response_schema`, Deepgram→Soniox).

**New findings (now in chapters 07/08/02e):** no `.dockerignore` (backend image includes `.env` and `node_modules`; local-build hygiene, not a repo leak); compose frontend on host port 3002 vs 3000 in README/CORS default; no client-side timeouts; the live agent LLM sets no thinking config; `validate_report` accepts out-of-range scores and a string `question_feedback`; measured Silero VAD about 4% of a core per stream and about 209 MB RSS per process with VAD loaded (single local measurement).

**Depth added:** all chapters expanded (about 78,000 words total, up from 35,000); interview bank 28 → 86 questions; exercises 11 → 20 with recorded outputs.

**Secrets check:** pattern scan for key/JWT-like strings found nothing; `.env` values were never read into docs.

**Remaining material gaps:** Mermaid never rendered; frontend only lightly audited; no live-session measurements (latency, report success, silent-turn rate); judge-calibration example uses invented labels (flagged); the Gemini quota is unknown; the interview bank cannot cover your actual contributions or the target company. **Not complete coverage.**

**Needs further investigation:** is the unreferenced `create_task` in `end_interview` lost at shutdown; does the Docker agent log the turn-detector fallback; first-time Piston runtime install vs the 30 s timeout; effect of two tabs on one session.

**Resolved during final check:** the one skipped test is the whole of `test_follow_up_cap.py` — it is skipped in the local backend venv because `livekit.agents` isn't installed there (`pytest -rs` **[Measured]**). CI installs `livekit-agents~=1.5`, so the follow-up-cap tests run there but did **not** run in my local 53-pass result. Anyone quoting "the cap is tested" should know I did not see those tests execute.
