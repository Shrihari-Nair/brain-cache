# 08 — Quick Reference

A revision aid. Each item points to the chapter that explains it properly.

## One-paragraph summary
MockMind turns a resume and a job description into a personalised voice interview. A **FastAPI backend** analyses the inputs with Gemini and stores an interview plan in **Redis**. The browser fetches a **LiveKit token** that also dispatches the **interviewer agent** into a room. The agent speaks with the candidate (speech-to-text → Gemini → text-to-speech), follows the plan in code, can show a coding question and review submitted code using a **Piston** sandbox, and finally produces a structured **report**. The backend and the agent never call each other; they communicate only through Redis. See [00](00-start-here.md) and [02](02-architecture.md).

## Components at a glance

| Component | Job | Fails → effect | Details |
|---|---|---|---|
| Frontend (Next.js) | setup form, live room UI, code editor, report page | no UI; backend/agent unaffected | [02](02-architecture.md) |
| Backend (FastAPI) | validate input, run preparation, mint tokens, serve/retry reports | no new interviews or reports | [02](02-architecture.md), [03](03-how-it-works.md) |
| Redis | shared state with expiry (plan, session, report, status, memory) | everything depends on it | [02](02-architecture.md) |
| LiveKit | real-time audio and data channel between browser and agent | no live interview | [01](01-foundations.md) |
| Interviewer agent | voice pipeline + question flow + report trigger | interview stops/derails | [03](03-how-it-works.md), [04](04-core-concepts.md) |
| Gemini | analysis, question generation, live conversation, code review wording, report | retried 3× then visible failure | [04](04-core-concepts.md) |
| Piston | sandboxed code execution | code review loses grounding | [03](03-how-it-works.md) |
| Eval harness | persona-driven simulated interviews, checks, judge | none at runtime | [06](06-quality-and-reliability.md) |
| Langfuse/OTel (optional) | traces of LLM calls | no traces only | [06](06-quality-and-reliability.md) |

## Data lifetimes (Redis)
Plan and session state ≈ 4 h · report and report status ≈ 24 h · candidate memory ≈ 1 year (opt-in, keyed by hashed email). Compose Redis keeps image-default snapshots only, so recent writes can be lost on a crash. See [02](02-architecture.md).

## Glossary

| Term | Meaning |
|---|---|
| Agent (LiveKit) | A worker process that joins a room as a participant and runs the voice pipeline |
| Agent dispatch | Instruction in the token that makes LiveKit send our named agent into the room |
| Constrained agent | An LLM with a few tools running inside a code-controlled flow ([04](04-core-concepts.md)) |
| Workflow (LLM) | Fixed sequence of LLM calls; the preparation phase |
| Structured output | Asking the model for JSON and validating it before use |
| Thinking budget | How many hidden reasoning tokens Gemini may use; they count against the output limit |
| Backoff | Waiting longer between retries (1 s, then 2 s here) |
| SSE | Server-Sent Events: one-way streaming over HTTP, used for preparation progress |
| WebRTC / LiveKit | Real-time audio/video technology / the service built on it |
| STT / TTS | Speech-to-text / text-to-speech |
| VAD | Voice-activity detection: is someone speaking right now? |
| Endpointing / turn detection | Deciding the speaker has finished their turn |
| Preemptive generation | Starting the reply before the turn is confirmed finished, to save latency |
| Data channel | Small messages sent over LiveKit (coding question out, code submission back) |
| Follow-up cap | At most 2 follow-ups per question; counted in code, enforced softly through a per-turn note |
| Prompt injection | Text in the input that tries to act as instructions to the model |
| Soft vs hard enforcement | Asking the model to comply vs making it impossible not to |
| Eval / judge | Measuring behaviour statistically / an LLM scoring a transcript against a rubric |
| Persona | A scripted simulated candidate used by the evals |
| Report state | `pending` → `ready` or `failed`; the UI polls and can trigger a retry |
| TTL | Time-to-live: automatic expiry of a stored key |

## Commands

| Purpose | Command | Status when I documented it |
|---|---|---|
| Backend tests | `cd mockmind && backend/.venv/bin/python -m pytest backend/tests -q` | **[Measured]** 52 passed, 2 skipped on a machine with no Redis (skips: the follow-up-cap file, which needs `livekit.agents`, and the Redis smoke test) |
| Same tests with agent libraries | `cd mockmind && agent/.venv/bin/python -m pytest backend/tests -q` | **[Measured]** 57 passed, 1 skipped (smoke test needs Redis) |
| Why was something skipped? | add `-rs` to pytest | **[Measured]** |
| Frontend checks | `cd mockmind/frontend && npx tsc --noEmit && npm run build` | in CI; not run here |
| Evals | `cd mockmind && python evals/run_eval.py --persona all --runs 3 --concurrency 3 --min-pass-rate 0.8` | in the manual workflow; **not run** (needs `GOOGLE_API_KEY`; coding personas need Piston) |
| Local stack | `cd mockmind && docker compose up` | defined in `docker-compose.yml`; not run |
| Health check | `GET /health` → `{"status":"ok"}` | static; does not probe Redis or Gemini |

Test counts depend on what is installed and whether Redis is reachable; quote them with that caveat. Compose publishes the frontend on host port 3002 while the README and the backend CORS default assume 3000 — check this if the browser reports CORS errors.

## Where to look when something breaks

| Symptom | First place to look | Explained in |
|---|---|---|
| Preparation fails | backend log for retry warnings; the LLM trace | [06](06-quality-and-reliability.md) |
| "Session not found" at join | the plan expired (4 h) or preparation never completed | [03](03-how-it-works.md) |
| Interview never starts | is the agent process running and registered with LiveKit? | [03](03-how-it-works.md) |
| Report stuck or failed | the report status key; use the retry endpoint | [03](03-how-it-works.md), [05](05-engineering-decisions.md) |
| Interviewer ignores the follow-up limit | expected occasionally (soft enforcement); run the evals | [04](04-core-concepts.md) |
| Candidate lost progress after Reconnect | by design today: Reconnect restarts the interview | [03](03-how-it-works.md) |

## Reading map
[00 Start here](00-start-here.md) → [01 Foundations](01-foundations.md) → [02 Architecture](02-architecture.md) → [03 How it works](03-how-it-works.md) → [04 Core concepts](04-core-concepts.md) → [05 Engineering decisions](05-engineering-decisions.md) → [06 Quality & reliability](06-quality-and-reliability.md) → [07 Interview preparation](07-interview-preparation.md).
The earlier, symbol-level notes (useful only when you need to find an exact implementation) are in `_archive-detailed-notes/`.
