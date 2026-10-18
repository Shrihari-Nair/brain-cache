# 11 — Interview Preparation

**Is this enough to attend the interview?** Short, honest answer: **this chapter is necessary, not sufficient.** It gives you code-accurate answers for about 100 questions about this system. It cannot give you (a) fluency — you must say the answers aloud until you stop reading them, (b) your personal contribution — the repo records one Git identity and does not say who diagnosed or built what, and (c) everything an interviewer may ask that is *not* about this codebase (see "Coverage and readiness" at the end). Use it as a script to practise from, not as something to memorise.

**How to use:** answer aloud first (60–90 s), then compare. Each entry has a model answer, `Source:` pointers (`path → symbol (L..)`, line numbers as of this checkout, all re-checked against the code), likely follow-ups, and common misconceptions. Evidence labels as in [00](00-learning-roadmap.md): **[Verified]** read in code, **[Measured]** I ran it, **[Library source]**, **[Standard knowledge]**, **[Inference]**, **[Hypothesis]**, **[Not measured]**.

**Ground rules for honesty (these protect you):**
1. Speak to what *you* did. Where an answer needs a personal claim I wrote `[your actual contribution]`. Do not fill it with something you did not do. "The project does X; I was responsible for Y" is only usable if Y is true.
2. Never quote numbers labelled [Not measured]. This system has **no latency, cost, load, accuracy or user numbers**. If asked, say so and say how you would measure.
3. "I haven't root-caused that; here is how I'd trace it" is a strong answer; an invented cause is a weak one.
4. If asked whether you built it alone or used AI assistance, answer truthfully. One commit message in this repo carries a `Co-Authored-By` trailer (`3d57ab0`); the repo does not otherwise say.

---

## The 60-second pitch (adapt to your real contribution)
"MockMind is an AI mock-interview platform. A candidate uploads a resume and a job description; the backend analyses both with Gemini and prepares a personalised question plan, streaming progress over SSE. The candidate then joins a LiveKit voice room where an agent — Soniox speech-to-text, Gemini, Cartesia text-to-speech — interviews them following that plan. For coding questions it shows an editor, runs the submitted code in a Piston sandbox and reviews it. At the end it produces a structured report. The backend and agent never call each other; they share state through Redis. We test with pytest with the LLM stubbed, and a persona-driven eval harness has caught real behavioural bugs, like the interviewer exceeding its follow-up limit. [Your actual contribution: one sentence.]"

## Model 2-minute walkthrough A — the architecture (~300 words, say it with the diagram from [03](03-system-architecture.md))
"There are four runtime pieces and two external services. The **frontend** is Next.js: a setup page, an interview page, a report page. The **backend** is FastAPI with three jobs: prepare an interview, mint a LiveKit token, and serve the report. The **agent** is a LiveKit Agents worker that runs one job per room: speech-to-text, Gemini, text-to-speech, with three tools — move to the next question, end the interview, get the current question. **Redis** is the integration bus: the backend writes the plan under `interview_plan:{id}`; the agent loads it when it joins; the agent writes the transcript and the report status; the backend serves them. Backend and agent never call each other — the contract is key names plus Pydantic models. **Piston** runs candidate code; only the agent calls it. **Langfuse** is optional tracing.

Two design ideas matter. First, the interview is a **workflow wrapped around a constrained agent**: preparation is a fixed pipeline — analyse resume, analyse job description, generate questions with a difficulty sequence assigned in Python — and the live interviewer is an LLM with three tools inside a code-driven question state machine. Second, **prompts are requests, code is enforcement**: the follow-up limit is counted in code and injected as a note every turn, and a deterministic eval check measures violations — but it is still a soft guarantee.

What I'd flag unprompted: there is no authentication, Redis has only default snapshot persistence (RDB, AOF off, anonymous volume), the report is generated inside the agent's shutdown path, and nothing about latency or load is measured."
*Sources behind each sentence:* `backend/routers/prepare.py → prepare_interview (L97)`; `backend/routers/session.py → get_token (L31)`; `agent/agent.py → entrypoint (L166)`; `agent/interviewer.py → InterviewerAgent (L44)`; `agent/session_state.py → follow_up_policy_note (L137)`.

## Model 2-minute walkthrough B — one end-to-end flow (~300 words)
"A candidate uploads a PDF and a job description and clicks start. The browser POSTs multipart form data to `/api/prepare`; the response is a server-sent-event stream. The backend validates type and size, extracts text with pypdf, then runs resume analysis and JD analysis — each one Gemini call that must return JSON, parsed and validated with Pydantic, retried up to three times with 1- and 2-second backoff. Then question generation: Python first assigns a difficulty to each slot — slot one is a fixed basic warm-up, the last slot is the closing question, the middle ones are drawn from weights — and optionally marks coding slots; the LLM fills them in. The plan is saved to Redis with a four-hour TTL, and the stream ends with `complete` and the session id.

The browser then calls `/api/token`. The backend checks the plan exists, then signs a LiveKit JWT that lets that browser join `interview_{id}` and carries an explicit agent dispatch with the session id as metadata. LiveKit starts a job on our worker; the worker reads the id from the metadata, loads the plan from Redis, builds the voice pipeline and speaks the opening message plus question one. Every candidate turn passes through a hook that counts follow-ups and injects a note; when the model is done it calls `move_to_next_question`. A coding question publishes the editor over a data channel; the submission comes back the same way, runs in Piston, and Gemini reviews it against the real test results.

On `end_interview` — or at shutdown — the agent saves the transcript to Redis and runs the report pipeline: status `pending`, one Gemini call, validation, store, status `ready` — or `failed`, which the report page surfaces with a retry button. If the candidate opted in to memory, scores and notes are merged into a per-email record."
*Sources:* `prepare.py L117–211`; `session.py L31–84`; `agent.py L166–274, L130`; `interviewer.py L82–156`; `report_pipeline.py L11`; `memory_updater.py L71`.

---

# Core questions (60)

## Level 1 — Fundamentals (10)

**Q1.1 What is the difference between an LLM *workflow* and an *agent*? Which is MockMind?**
A workflow is a fixed code path in which LLM calls are steps; an agent lets the model choose actions in a loop. Preparation is a workflow (analyse resume → analyse JD → generate questions). The live interviewer is a **constrained agent**: Gemini with exactly three tools inside a code-controlled question sequence — the code decides what the "current question" is, the model decides how to ask, whether to follow up and when to call `move_to_next_question`. There is no autonomous planning, no multi-agent orchestration, no retrieval/embeddings.
Source: `agent/interviewer.py → InterviewerAgent (L44)`, tools L82/L138/L159; [02b](02b-ai-and-llm-deep-dive.md).
Follow-ups: "Where is the autonomy limited?" (question order and counters live in `AgentSessionState`; the model cannot skip or reorder.) "Is it multi-agent?" Misconception: the resume/JD/question analyzers are "sub-agents" — the code comments say sub-agent, but each is one sequential call with no loop or tools.

**Q1.2 What is stored in Redis, with what TTLs, and who writes it?**
Five key families. `interview_plan:{id}` (backend, 4 h) and `session_state:{id}` (backend writes an empty one at prepare, the agent overwrites it with transcript/progress, 4 h, refreshed on every save), `interview_report:{id}` and `interview_report_status:{id}` (pipeline, 24 h each), and `candidate_memory:{sha256(email)}` (1 year). Redis is a shared TTL store here, not a database. If it restarts, everything is gone — the compose file has no volume for it.
Source: `backend/routers/prepare.py L186/L194`; `agent/session_state.py → save (L112)`; `backend/services/report_store.py L8–9`; `backend/services/memory_store.py L6–12`. I listed every `setex/get/delete` by grep (exercise 1.2).
Follow-ups: "What if Redis restarts mid-interview?" (the autosave and report fail, logged only — D12.) Misconception: "the memory key is anonymous" — the value contains the plain email (`CandidateMemory.email`); only the *key* is hashed.

**Q1.3 What is SSE and why use it for `/api/prepare`?**
Server-Sent Events: one long-lived HTTP response with `Content-Type: text/event-stream` carrying `data: {json}\n\n` records, server → client only. The endpoint streams progress (5 → 15 → 20 → 45 → 50 → 65 → 70 → 90 → 100) through the three LLM steps so the UI isn't blank for tens of seconds. The browser reads it with `fetch` plus a stream reader rather than `EventSource`, because the request is a multipart POST (`EventSource` only issues GET [Standard knowledge]). There are no event ids, so a dropped stream cannot resume and the hook has no "stream ended without complete" timeout (D6).
Source: `prepare.py → event_stream (L117), _sse (L228)`; `frontend/hooks/usePreparation.ts` (fetch L51, parsing L60–95). Offline run of the stream: exercise 2.6.
Misconception: "SSE is WebSocket-lite" — it is plain HTTP, one direction.

**Q1.4 What does `asyncio` buy you here, and where does it not help?**
It lets one thread overlap I/O waits: while one request awaits Gemini or Redis the loop serves others. It does not help blocking or CPU work: resume parsing (`pypdf`/`python-docx`) is synchronous and runs directly in the async generator, so a large PDF stalls the loop for its duration. Impact is [Not measured]. A fix is `asyncio.to_thread` or a process pool. Also note the agent opens a **new Redis connection on every `save()`** rather than reusing a client.
Source: `prepare.py L129` → `services/resume_parser.py → extract_text_from_pdf (L5)`; `agent/session_state.py → save (L112)`.
Follow-up: "How would you measure the stall?" (event-loop lag metric while uploading a big PDF.)

**Q1.5 Why validate LLM output with Pydantic?**
The model returns text that is "probably JSON". Validation turns it into typed data and fails loudly: for example `category` is an enum, so `"coding"` raises. Failures flow into `retry_async`, which re-asks. Three layers exist: JSON mime type on the request, `json.loads(_clean_json(...))`, then Pydantic models or `validate_report`. The report validator is hand-written, not a Pydantic model, and (I ran it) accepts `overall_score=99` and `nan`.
Source: `backend/models/interview_plan.py → QuestionCategory (L6), InterviewQuestion (L33)`; `backend/services/report_generator.py → validate_report (L12)`. Run: exercise 2.1.
Misconception: "JSON mime type guarantees the schema" — it guarantees syntactic JSON, not your keys or ranges.

**Q1.6 What are WebRTC, an SFU and LiveKit, in two sentences each?**
WebRTC is the browser-native stack for real-time audio/video/data, with NAT traversal (ICE/STUN/TURN) and encrypted media (SRTP). An SFU receives each participant's stream and forwards it without mixing. LiveKit is a platform built on WebRTC: hosted SFU, client/server SDKs, and the Agents framework for running AI participants in rooms. Our code mints tokens, defines the agent and uses data topics; LiveKit does signalling, media and dispatch.
Source: [02c](02c-real-time-voice-and-livekit.md) §2c.1 (**[Standard knowledge]** plus code mapping); `backend/routers/session.py L51–76`.

**Q1.7 What is a cascaded voice pipeline and why this choice over speech-to-speech?**
STT → LLM → TTS, orchestrated by `AgentSession`. Transcripts become first-class: needed for the follow-up counter, the report, the retry endpoint, tracing and evals; tools are mature; vendors are swappable. Speech-to-speech models cut latency but give less text-level control. The original spec states the "half-cascade" rationale (text-only Gemini + Cartesia for voice quality and control) **[Verified: spec §13 Note 1 per 02c]**. Latency of this pipeline is [Not measured].
Source: `agent/agent.py L200–215` (`soniox.STT`, `google.LLM`, `cartesia.TTS`, `silero.VAD`).

**Q1.8 How does the browser prove it may join a room, and why can't it forge access?**
The backend signs a JWT (HS256) with `LIVEKIT_API_SECRET`. Claims include `video.roomJoin`, the single allowed room `interview_{id}`, publish/subscribe/publish-data permissions, identity `candidate_{id[:8]}`, and `roomConfig.agents` with the dispatch. The secret never leaves the server, so a client cannot create a valid signature. TTL is 2 hours. I decoded a token minted by the real handler with a dummy key to confirm these claims (exercise 2.5).
Source: `session.py → get_token (L31)`, plan check L37–42, JWT L51–76, TTL L55.
Misconception: "the token is encrypted" — a JWT is signed, not encrypted; anyone holding it can read the claims.

**Q1.9 What does an `InterviewPlan` contain?**
`candidate` (profile from the resume), `job` (profile from the JD), `questions[]`, `opening_message`, `total_duration_minutes`, `created_at`, `level_weights`, optional `candidate_email`. Each `InterviewQuestion` has id, category, question text, intent, `ideal_answer_points` (the answer key), `follow_ups` (trigger + question), difficulty 1–3, source, `question_type` (`conversational|coding`) and an optional `coding_spec` (language, problem, starter code, test cases).
Source: `backend/models/interview_plan.py L6–94`.
Follow-up: "Where does the answer key go?" Into the live system prompt (the full plan JSON), which is a leakage risk (C8) and costs input tokens every turn.

**Q1.10 Name the stack and what each piece does.**
Frontend: Next.js (App Router) + TypeScript + Tailwind + `livekit-client`. Backend: FastAPI/Uvicorn, Pydantic, `google-genai`, `pypdf`, `python-docx`, `livekit-api`, `redis.asyncio`, Langfuse. Agent: `livekit-agents` with Soniox, Cartesia, Silero plugins and Gemini. Infra: Redis 7, Piston (privileged container), optional self-hosted Langfuse, Docker Compose. Model: `gemini-2.5-flash` for every call site.
Source: `backend/requirements.txt`; `mockmind/docker-compose.yml`; `agent/agent.py L200–215`; [02a](02a-backend-and-data-technologies.md)–[02e](02e-infra-and-tooling.md).
Misconception: "Deepgram is used" — the original spec said so; the code uses Soniox (`agent.py L202–206`).

---

## Level 2 — Implementation (10)

**Q2.1 Walk me through `retry_async`.**
It runs a zero-argument async callable up to `attempts=3`. On any `Exception` it logs a warning and sleeps `base_delay * 2**i` (1 s then 2 s) before the next try; after the last failure it logs an error and re-raises the *last* exception. Retrying *every* exception is deliberate: a malformed JSON response is often fixed by re-asking. There is no jitter, circuit breaker or error classification, so a quota error is retried like a parse error. I confirmed 3 calls, sleeps `[1.0, 2.0]` and `ValueError('bad 3')` offline (exercise 1.1).
Source: `backend/services/llm_retry.py → retry_async (L10–33)`; `test_llm_retry.py`.
Follow-ups: "What would you change?" (retry only transient classes, jitter, a budget, a breaker.) "Is the wrapped call safe to repeat?" (yes for pure generations; the eval driver needed a snapshot/rollback because its tools mutate state.) Misconception: "it retries forever".

**Q2.2 How is the difficulty mix chosen, and is it guaranteed?**
In Python, before the LLM sees anything: weights (default 0.40/0.35/0.25) are normalized; slot 0 is always "basic" (warm-up), the last is the closing question with no level, and every middle slot is drawn with `random.choices`. Coding slots are chosen with `random.sample`, preferring non-basic slots. Then the prompt lists the sequence and asks for exactly one question per slot. It is therefore a **random draw per interview, not an exact quota**, and the model's compliance is only *logged* (target vs actual counts), not validated or retried. I measured 10 000 draws for 16 questions: the count of "advanced" varied from 0 to 11 with a mode of 3 (exercise 2.2). The README's "guarantee" wording is overstated.
Source: `question_generator.py → _normalize_level_weights (L209), _assign_level_sequence (L224), _assign_coding_slots (L239), _build_prompt (L254)`; level logging in `_generate_questions_once (L297)`.
Misconception: "weights 0.4/0.35/0.25 mean 6/5/4 questions".

**Q2.3 How does the follow-up cap work, and is it enforced?**
`MAX_FOLLOW_UPS = 2`. On every completed candidate turn the agent's `on_user_turn_completed` hook calls `follow_up_policy_note(state)`, which calls `record_candidate_turn()` (first turn = the answer; each later turn = a follow-up) and returns a system note: `Follow-ups used on this question: n/2.` — plus, at the cap, "do NOT ask another follow-up… call move_to_next_question". The hook adds it to the turn context. Coding questions return `None` (no counting). It is **soft**: nothing in code blocks the model from asking another; the eval's deterministic check measures violations afterwards. Note the hook's own docstring says "enforces the follow-up cap in code" — the accurate description is "counts in code, requests via a note". Because the count is derived from turns, the display can read `3/2` if the model ignores the stop note (I observed this offline, exercise 1.4).
Source: `agent/interviewer.py → on_user_turn_completed (L70)`; `agent/session_state.py → record_candidate_turn (L81), follow_up_policy_note (L137)`; `evals/checks.py L35`.
Follow-up: "Why not auto-advance in code?" (naturalness vs guarantee, [10](10-design-decisions-and-tradeoffs.md) #6.) Misconception: "the code prevents a third follow-up".

**Q2.4 How is the report produced, and what happens if it fails?**
`generate_and_store_report` first saves the final state (so the transcript survives), then `run_report_pipeline`: set status `pending`, call `generate_report` (Gemini JSON, temperature 0.2, up to 3 attempts, `validate_report`), store the report (24 h) and set `ready`; on any exception it sets `failed` and returns `False` — it never raises. If the candidate opted in, memory is then updated; that part is wrapped separately so a memory failure cannot mark the report failed. The report page polls `GET /api/report/{id}`: 200 report, 404 not ready, 502 with `status: failed` → shows Retry, which `POST`s `/retry`.
Source: `agent/session_state.py → generate_and_store_report (L158)`; `backend/services/report_pipeline.py → run_report_pipeline (L11)`; `routers/session.py → get_report (L88), retry_report (L113)`; offline run of both outcomes: exercise 2.7.
Follow-up: "What if the process dies mid-generation?" — `pending` stays up to 24 h with no staleness check (D3, [Hypothesis]).

**Q2.5 How does the agent end up in the right room?**
The token endpoint checks `interview_plan:{id}` exists (404 otherwise), then embeds `RoomConfiguration(agents=[RoomAgentDispatch(agent_name="mockmind-interviewer", metadata={"session_id": …})])`. When the browser connects, LiveKit sees the dispatch and assigns a job to a worker registered under that name; `entrypoint` reads the session id from the job metadata. Dispatch is **explicit**: an agent only joins rooms that request it.
Source: `session.py L37–42, L65–74`; `agent/agent.py → entrypoint (L166)`, `@server.rtc_session(agent_name=…)` L165.
Failure details: a missing `LIVEKIT_*` env var raises `KeyError` (HTTP 500) — I reproduced it (exercise 2.5); a missing session id makes the job log an error and return.
Misconception: "the agent auto-joins every new room".

**Q2.6 How is submitted code reviewed?**
The browser sends `{question_id, code}` on the data-channel topic `code_submission`. The agent's `on_data_received` spawns `_handle_code_submission`, which finds the question, then `review_code`: for each test case it runs `candidate_code + "\n\n" + test.input` in Piston (the `input` is a Python snippet that *calls* the function and prints; it is not stdin), compares stripped stdout to `expected_output` and exit code 0, then asks Gemini for a JSON review given the *actual* results. Finally the code **overwrites** the model's `tests_passed`/`tests_total` with the counted ground truth. The result is recorded in the transcript and fed to `session.generate_reply`.
Source: `agent/agent.py → _handle_code_submission (L130), on_data_received (L246)`; `services/code_judge.py → review_code (L70)`; `services/code_executor.py → execute_code (L49)`; `agent/code_review_flow.py L14/L23`.
Follow-up: "What if Piston is down?" — each test's exception becomes `passed=False` with `stderr: execution error`, so correct code looks failing (D7). Note the model docstring says test `input` is "passed to stdin" — stale; the code appends it.

**Q2.7 How are inputs validated, and how do you avoid leaking internals?**
Resume: extension must be pdf/docx/doc/txt, size >0 and ≤ 5 MB (the endpoint reads `MAX_RESUME_BYTES + 1` so oversize is detected without buffering a huge upload); extracted text is truncated to 30 000 characters; the JD is capped at 15 000 characters. Failures raise `PreparationError`, whose message is safe to show; any other exception is logged in full and the client sees only `GENERIC_PREP_ERROR`. A test proves a secret in an exception message never appears in the stream. Fixed *presets* (not raw numbers) mean the client cannot set weights or counts.
Source: `prepare.py L56–78, L117–135, L213–223`; `test_prepare_options_and_limits.py`.
Misconception: "extension checking is security" — it is a usability check; content is not sniffed.

**Q2.8 How is the transcript captured and persisted?**
Two event handlers: `user_input_transcribed` (final results only) adds candidate entries, `conversation_item_added` adds assistant entries (de-duplicating the opening line already added in `on_enter`). Each entry is `{speaker, text, timestamp, question_index}`, where `question_index` is the state's index **at write time**. `add_transcript_entry` schedules a debounced autosave: the first entry starts a task that sleeps 1 s and then saves everything, so a burst of entries yields one Redis write (a test asserts 5 entries → 1 save). A long answer may be several final STT segments → several entries.
Source: `agent/agent.py L224–243`; `agent/session_state.py → add_transcript_entry (L69), _schedule_autosave (L93), _autosave (L105)`; `test_follow_up_cap.py::test_transcript_autosave_coalesces_writes`.
Follow-up: "What is lost on a crash?" — up to ~1 s of transcript plus anything unsaved.

**Q2.9 How does candidate memory update after a session?**
After the report succeeds and only if `plan.candidate_email` is set: append the session id, append each category score to `category_scores_history`, recompute `weak_categories` (categories whose **average over all history** is below 6.0, worst first, max 3), append this plan's questions (keeping the last 50), then regenerate a 2–3 sentence trend note with Gemini (temperature 0.3, thinking off). The merge is a pure function; Redis I/O wraps it. Subtlety I verified by running it with a stubbed note generator: one bad session (4.0) then a good one (9.0) averages 6.5 → "technical" is no longer weak after two sessions (exercise 3.2).
Source: `services/memory_updater.py → update_memory_from_report (L71), _recompute_weak_categories (L37)`, constants L11–13; `report_pipeline.py L38–53`.
Follow-up: concurrent opted-in sessions can lose an update (read-modify-write, C18).

**Q2.10 What does `validate_report` enforce, and why is it hand-written?**
It requires `overall_score`, `hiring_recommendation`, `category_scores`; the recommendation must be one of `strong_yes|yes|maybe|no` (the report page indexes a config map by it, and an unknown value used to crash the page); `category_scores` must be an object; `overall_score` is coerced to float; missing `summary/strengths/improvements/question_feedback` are defaulted. It does **not** check score ranges, `nan`, or the shape of `question_feedback` (all accepted when I ran it). A raise inside it is caught by `retry_async`, so a bad report gets regenerated.
Source: `report_generator.py → validate_report (L12)`; `test_report_validation.py`; run: exercise 2.1.

---

## Level 3 — Architecture (10)

**Q3.1 Draw the system and name the trust boundaries.**
Browser ↔ backend (HTTP/SSE, unauthenticated), browser ↔ LiveKit (WebRTC, token-gated), LiveKit ↔ agent worker (registered with key/secret), backend/agent ↔ Redis (no auth in compose), agent ↔ Piston (no auth, privileged container), backend/agent ↔ Gemini/Soniox/Cartesia (API keys), services ↔ Langfuse (optional). The backend and agent never call each other. The sensitive data in flight: resume/JD text (to Gemini and into traces), audio (to LiveKit, Soniox, Cartesia), candidate code (to Piston).
Source: [03](03-system-architecture.md); `docker-compose.yml`.

**Q3.2 What are the single points of failure?**
Redis (single node, snapshot-only persistence — writes since the last snapshot can be lost: plans, transcripts, reports, memory), the Gemini dependency (every step), the agent process for a live session (no resume — "Reconnect" restarts the interview, C1), and LiveKit Cloud. Report generation inside the agent's end-of-session path is a reliability weak point (D1, [Hypothesis]).
Source: `docker-compose.yml` (no volume on `redis`); `AgentSessionState.load` (`session_state.py L32`) reads only the plan.

**Q3.3 How would you scale to 100 concurrent interviews?**
Reasoned only — **no load numbers exist**. The backend is stateless: scale out behind a load balancer (the only in-process state is `_retry_tasks`). The agent is the constraint: one job process per room; add workers; LiveKit load-balances by reported load [Library source: `load_threshold` 0.7]. Gemini quotas need a limiter and backoff with jitter. Redis needs HA and persistence. Measure first: per-job CPU/memory and turn latency under a ramp (exercise 3.4).
Source: `agent/agent.py` (no `prewarm`, no `ServerOptions`); [08](08-performance-reliability-and-scaling.md).

**Q3.4 Why prepare a plan before the call instead of one live prompt?**
Lower live latency (no analysis during the call), a consistent, inspectable structure (the plan can be tested and evaluated), and per-slot control of difficulty and coding slots in Python. Costs: an upfront wait, a plan that can go stale (4 h TTL), and the answer key sitting in the live prompt.
Source: [10](10-design-decisions-and-tradeoffs.md) #4; `prepare.py L137–183`.

**Q3.5 There is no auth. What would you do?**
Today a `session_id` (UUIDv4) is a capability: whoever has it can mint a token, read the report and retry it. Add user accounts or signed short-lived session links, bind the session id to the user, authenticate the retry/token/report endpoints, rate-limit `/api/prepare` (each call spends LLM money), and put Redis behind auth/TLS. CORS in `main.py` limits browsers only, not scripts.
Source: `backend/main.py L21–29`; C3.
Misconception: "CORS protects the API".

**Q3.6 What is the actual contract between backend and agent, and what are its risks?**
Key names and JSON shapes: `interview_plan:{id}` (`InterviewPlan`), `session_state:{id}` (`SessionState`), report/status keys, and the agent dispatch metadata `{"session_id": …}`. The agent **imports the backend package** for the models and `report_pipeline`, so the schema cannot drift, but the agent image needs backend code. A rename of a key is a silent cross-process break; only the smoke test exercises both sides.
Source: `agent/session_state.py L32–45`; `agent/interviewer.py L9–15`; `tests/test_api_smoke.py`; [10](10-design-decisions-and-tradeoffs.md) decisions 1–2.

**Q3.7 Where does personal data live, and who sees it?**
Resume text and JD → Gemini prompts → Langfuse traces (full prompts, no redaction, no retention policy, C15). Plan (incl. `candidate_email` if opted in) → Redis 4 h. Transcript/report → Redis 4 h/24 h. Memory (plain email inside the value, name-less scores/notes) → Redis 1 year, with a `DELETE /api/candidate-memory` endpoint that is itself unauthenticated. Audio → LiveKit, Soniox, Cartesia (third-party processors).
Source: `services/tracing.py L44–72`; `memory_store.py`; `routers/memory.py L13`.
Follow-up: "Is hashing the email enough?" — no: guessable emails can be hashed by an attacker; and the email is also stored in the value [Inference].

**Q3.8 How does the system degrade when optional parts fail?**
Memory fetch failure → no memory, preparation continues (`_maybe_fetch_memory` swallows; tested). Memory save failure → logged, report already stored. Tracing keys absent → no-op. Turn-detector model missing → `_load_turn_detector` returns `None`, VAD-only. Report failure → status `failed` + retry. Not degraded: Redis down at startup (lifespan `ping` raises), Gemini down (3 tries then a generic error).
Source: `prepare.py L81–93`; `report_pipeline.py L38–53`; `tracing.py L58–60`; `agent.py L119–127`; `main.py L15–18`.

**Q3.9 Walk through the deployment topology.**
`docker-compose.yml` runs `redis`, `piston` (privileged, with a packages volume and a tmpfs), `backend` (uvicorn `--reload`, source mounted), `agent` (`python agent/agent.py start`), `frontend` (host port 3002), and the Langfuse stack. Backend and agent get `REDIS_URL=redis://redis:6379` and `PISTON_URL`. The reload/mount setup is a development configuration; a production topology is not defined in the repo.
Source: `mockmind/docker-compose.yml`; [02e](02e-infra-and-tooling.md).

**Q3.10 Why did the agent import the backend package, and what does that couple?**
Reuse of `InterviewPlan`/`SessionState`/`run_report_pipeline`/`code_judge` without duplicating them. Coupling: the agent container needs the backend source and dependencies (`google-genai`, `redis`, Langfuse), and a backend change deploys an agent change. The alternative — a published shared package, or HTTP between them — adds ceremony; the repo records no rationale for the choice **[Not recorded]**.
Source: `agent/session_state.py L7`; `agent/agent.py L11–` imports; [10](10-design-decisions-and-tradeoffs.md) #2.

---

## Level 4 — Debugging (10)
For each, state the evidence level; I mark what is observed versus hypothesised.

**Q4.1 "Question generation sometimes crashes." (A1, commit evidence)**
Read the traceback: `QuestionCategory("coding")` raised. Reproduce by constructing `InterviewQuestion(category="coding")` (validation error on `category`). Hypothesis from the commit message: the prompt always explained the coding JSON format even with zero coding slots, priming the model to put `"coding"` into `category`. Fix: include the coding block only when coding slots exist, and always state "category is never coding". Keep retry and validation as the backstop. The one offline regression test asserts the prompt assembly; the live 3-call check is a script.
Source: commit `b2ba44e`; `question_generator.py → _CODING_FORMAT_BLOCK (L186)`, `QUESTION_GENERATION_PROMPT (L127)`; `test_question_generator_coding_category_bug.py`.
Follow-up: "Why doesn't the negative rule alone suffice?" (it can't guarantee; removing irrelevant concepts reduces priming.)

**Q4.2 "Report JSON is unterminated." (A3)**
Check `finish_reason`: `MAX_TOKENS`. Hidden thinking tokens count against `max_output_tokens`, so the visible JSON was cut. Fix: thinking off, JSON mime type, larger ceiling (16 384), plus `validate_report` + retry + `failed` status. Remaining risk: the report copies every candidate answer, so stored reports are large (29–72 k characters in my stored files **[Measured]**).
Source: `evals/REPORT.md`; `report_generator.py L92–97`; [07](07-engineering-challenges.md) A3/C6.
Misconception: "raise the limit and done" — with thinking on, a higher limit just shifts the problem.

**Q4.3 "The interviewer asked too many follow-ups." (A5)**
An eval persona (`vague`) plus a judge exposed three follow-ups on one question and a leaked hint. Root cause: the rule lived only in the prompt, and `increment_follow_up_count()` existed but was never called. Fix: count candidate turns in code (`record_candidate_turn`), inject the count and a stop instruction every turn, and add a deterministic check. It is still soft, and the hook invalidates preemptive generation **[Library source]**, cost [Not measured].
Source: `session_state.py L81–91, L137–160`; `checks.py L35`; `test_follow_up_cap.py`.

**Q4.4 "A fresh clone doesn't start." (A6)**
Imports of `backend.services.tracing` at module scope failed because the file and requirements had never been committed. Verify with a clean checkout, not your working tree. CI (added later) installs from `requirements.txt` and runs tests on a clean runner, which would catch it.
Source: commit `7e98c15` message; `.github/workflows/ci.yml L33–51`.

**Q4.5 "The interviewer goes silent after a tool call." (B1)**
Observed in stored eval results: 2 of 10 checked `strong` runs failed `interviewer_never_silent` **[Measured]**. Root cause **unknown**. Hypothesis: the second completion after the tool result sometimes has no text. Note the eval interviewer is the plain google-genai chat with automatic function calling, not the LiveKit plugin, so the production behaviour is unverified. How I'd trace: log `finish_reason` and parts for empty responses; compare with the production path; check Langfuse spans of `interviewer_turn`.
Source: `evals/interview_driver.py L21, L170–180`; `checks.py L27`. Say plainly: "measured, not root-caused, not fixed."

**Q4.6 "The report's overall score contradicts the per-question scores." (B3)**
Two of 14 stored reports **[Measured]**: overall 9.5 with a per-question mean of 1.88. The prompt asks the model to emit overall, category and per-question scores independently. Fix direction: derive overall/category scores in code from per-question scores (or validate consistency and retry). Memory's weak-area logic reads `category_scores`, so inconsistent scores propagate.
Source: `report_generator.py L31–69`; [07](07-engineering-challenges.md) B3/C7.

**Q4.7 "Reports are missing after a call." (scenario; mostly D-class hypotheses)**
Check `interview_report_status:{id}`: absent → pipeline never ran (candidate left before the agent started it, or the job process was torn down); `pending` for a long time → the task died mid-generation (no staleness check, D3); `failed` → hit Retry (needs a non-empty persisted transcript, else 409). Subtle code fact: `end_interview` sets `ended_at` and spawns the report task, and the shutdown callback runs only `if not state.ended_at` — so if the process is torn down before that task finishes, nothing regenerates. Not observed in practice; success rate [Not measured].
Source: `interviewer.py L146–149`; `agent.py L251–257`; `session.py L113–139`.

**Q4.8 "The UI is stuck on Connecting…" (voice start)**
In order: is the agent worker running and registered? Did the token request 404 (plan expired after 4 h) or 500 (missing LiveKit env)? Does the job log "No session_id in job metadata"? Was the dispatch metadata set? Is the browser mic permission granted? The UI shows a "taking too long" overlay after 25 s (and may misread an `idle` agent state, D11 [Inference]).
Source: `session.py L37–46`; `agent.py L166–195`; [02c](02c-real-time-voice-and-livekit.md) §2c.13.

**Q4.9 "Correct code is reported as failing every test." (D7)**
Likely Piston unreachable or the language runtime not installed: `execute_code` raises, `review_code` catches per test and records `passed=False`, `stderr="execution error: …"`, and still calls Gemini with those results. Check `PISTON_URL`, `GET /api/v2/runtimes`, container logs. The code never distinguishes infrastructure errors from wrong answers (D7), so the candidate hears a failing review. In evals `preflight.check_piston` fails fast for exactly this reason.
Source: `code_judge.py L70–110`; `code_executor.py L17–49`; `evals/preflight.py L6`.

**Q4.10 "The eval pass rate is 1/3 — is the interviewer broken?" (B4/B5)**
Not necessarily. Look at *why* each run failed: the stored failures split into a rubric/prompt contradiction (the judge penalised the required AI introduction), a silent turn, and one run where the interviewer answered its own questions. With n=3, one pass can be luck (see Q6.6). Check the judge notes and the deterministic checks separately, fix the rubric if it contradicts the prompt, then re-run with more runs.
Source: `evals/results/` (local, git-ignored); `evals/judge.py L15–46`; [07](07-engineering-challenges.md) B4–B5.

---

## Level 5 — Trade-offs (10)

**Q5.1 Prompt rule vs code enforcement?** Prompt rules are free, natural, and unreliable; code rules are reliable and rigid. The project is hybrid for the follow-up cap: code counts and *tells* the model, an eval *measures*. A hard guarantee would be code-driven advance or an output validator, trading naturalness and latency. For coding slots and the editor, the code does it deterministically (`_publish_coding_challenge` is called from Python, not left to the model), the same lesson applied. Source: `interviewer.py L20–25, L101–102`; `session_state.py L137`.

**Q5.2 Thinking on or off?** Per task, evidence-driven: *off* for extraction, report, code review, memory notes (speed, truncation risk — A3); *on (dynamic, `-1`)* only for coding plans, where per-slot tagging dropped coding slots with thinking off (A2). Costs: slower/costlier coding preparation and a larger output ceiling. The "13/13" and "0 misses in 6" figures come from the commit/comment; I did not re-measure. Source: `question_generator.py L321–332`; `report_generator.py L96`.

**Q5.3 Redis or Postgres?** Redis: zero-ops, TTL-based expiry (privacy-friendly), simple JSON blobs. Postgres: durability, queries, transactions. For a pilot Redis is defensible; for reports and memory in production I'd move durable data to a database and keep Redis for ephemeral plan/session state. A concrete pain: only default RDB snapshots (no AOF), so a crash can lose recent writes and the anonymous volume is easy to lose; and memory updates are unlocked read-modify-write. Source: `docker-compose.yml`; `report_pipeline.py L43–51`.

**Q5.4 Generate the report in the agent or the backend?** In the agent: the transcript is already in memory, no extra transport. Cost: it runs in the end-of-session path, where **[Library source]** timeouts exist, and a failure has no HTTP caller — which is why a status key and retry endpoint were added. A backend worker triggered by the last autosave plus an explicit end signal would be sturdier. Success rate in live use is [Not measured]. Source: `agent.py L251–257`; `interviewer.py L149`; [10](10-design-decisions-and-tradeoffs.md) decision 11.

**Q5.5 JSON-in-prompt plus parsing vs native structured output (`response_schema`)?** Current: JSON mime type, `_clean_json` (duplicated five times), Pydantic or hand validation, retry. Pros: portable across prompts, simple. Cons: brittle (fences, stray text), duplicated helpers, a retry costs a full call, schema lives in prose. A response-schema approach would reduce parse failures, but values still need semantic validation (ranges, consistency). The repo records no reason for the choice. Source: `question_generator.py L13`, `resume_analyzer.py L14`, `jd_analyzer.py L14`, `code_judge.py L61`, `evals/judge.py L61`.

**Q5.6 A per-turn hook vs latency?** The hook gives the model visible state each turn. **[Library source]** the framework discards the preemptive reply if the context changes; adding a system message changes it. So the cap note costs the speculative head start. Magnitude [Not measured]. Alternatives: bake the counter into tool results (no context change on a normal turn but the model isn't told), or only add the note when the count changes. Source: `interviewer.py L70–79`; [02c](02c-real-time-voice-and-livekit.md) §2c.7.

**Q5.7 Cascaded pipeline vs a realtime speech-to-speech model?** Cascaded gives text at every stage (transcripts for reports/evals, tools, swappable vendors) at the price of extra hops (VAD/endpointing wait, STT finalisation, LLM first token, TTS first audio). Speech-to-speech cuts hops but reduces text-level control and observability. Per-stage latencies are [Not measured]. Source: `agent.py L200–215`; [02c](02c-real-time-voice-and-livekit.md) §2c.3, §2c.12.

**Q5.8 One model (Gemini 2.5 Flash) for everything?** Simple, fast, cheap-ish [pricing not verified here]. The repo records no reason for Flash over other models. Risks: no fallback model, a single-vendor quota, a lower ceiling on judgement, and — for the eval judge — the same family judging its own outputs. A mixed setup (small for extraction, larger for report/judge) is a plausible improvement requiring measurement. Source: grep of `model="gemini-2.5-flash"` across `backend/`, `agent/`, `evals/`.

**Q5.9 SSE vs WebSocket vs job-id + polling for preparation?** SSE: simple, one-way, native streaming, no extra server. Polling with a job id is resumable and survives reconnects (Redis would hold progress). WebSocket is overkill for one-way progress. The weakness of the current choice is no resume (D6) and no heartbeat during long Gemini retries. Source: `prepare.py L117`; `usePreparation.ts L51–106`.

**Q5.10 Custom eval harness vs a third-party eval tool?** Custom: shaped to this app (personas, tool mirrors, coding flow, deterministic checks, reuse of the real code paths). Third-party: dashboards, datasets, judge tooling. Costs of custom: hand-mirrored tools that can drift, an uncalibrated judge, one fixture. The harness reuses real production functions for analysis, generation and report, which is its main strength. Source: `evals/REPORT.md`; `evals/interview_tools.py L1–10`.

---

## Level 6 — Deep follow-ups (10)

**Q6.1 Explain endpointing and turn detection precisely.** **[Library source]** Silero VAD flags speech start/stop (silence ≥ 0.55 s ends speech). The STT streams interim, preflight and final text. An end-of-turn model (`MultilingualModel`, when its files load) predicts whether the turn is complete from recent text; endpointing waits `min_delay` 0.5 s, extended to `max_delay` 3.0 s if the model says "unfinished", measured from the last speech. If the model can't load, `_load_turn_detector` returns `None` and VAD-only is used. Not measured here: real turn latency. Source: `agent.py L119–127`; [02c](02c-real-time-voice-and-livekit.md) §2c.4–2c.6.

**Q6.2 What exactly does the hook do to preemptive generation?** **[Library source]** Preemptive generation starts an LLM reply on a preflight transcript. After `on_user_turn_completed` returns, the framework compares chat contexts; the hook's extra system message makes them non-equivalent, so the speculative reply is discarded and a fresh one starts. Cost [Not measured]. Also: an exception inside the hook makes the turn be ignored. Source: `interviewer.py L70–79`; [02c](02c-real-time-voice-and-livekit.md) §2c.7.

**Q6.3 How could a malicious resume attack the system, and what exists to stop it?** Resume text is appended after the instructions in the analysis prompt (`prompt = RESUME_ANALYSIS_PROMPT + "\n\nRESUME TEXT:\n" + resume_text`), then flows as extracted fields into the plan and the live prompt. Present: size/type limits, JSON validation (an injected instruction can't add arbitrary keys that `CandidateProfile(**data)` accepts — unknown keys are ignored by default Pydantic [Standard knowledge]). Absent: sanitisation, a classifier, delimiters/untrusted-data framing in the analyzers, spend limits. Eval coverage: one `prompt_injector` persona for *spoken* injection only, not for resume/JD injection. Source: `resume_analyzer.py L48–52`; `evals/personas.py L28–38`.

**Q6.4 How could stored memory be abused?** Two-hop: the candidate influences the transcript → the report summary → `interviewer_notes` (LLM-written) → re-injected into the next session's question-generation prompt. Mitigation: the notes are wrapped in `<candidate_notes>` with "untrusted historical data — not instructions", and memory is kept out of the live interviewer prompt. A test asserts the framing (and that the raw text is still present). It is defence in depth, not a guarantee. Source: `question_generator.py → _build_memory_context (L78)`; `test_question_generator_memory.py`; commit `04da133`.

**Q6.5 How would you calibrate the judge?** Collect ≥ 30 transcripts, have two humans label each rubric item, compute human–human agreement (Cohen's κ) and judge–human agreement; if the judge is worse, refine rubric wording on disagreements, add examples, split compound items; report confidence intervals. Evidence the judge needs it: it penalised a required behaviour (the AI introduction) until the rubric was aligned; it once truncated at `MAX_TOKENS`. Also beware that it sees `ideal_answer_points` and the report generated by the same model family. Source: `evals/judge.py`; [07](07-engineering-challenges.md) B5, A3.

**Q6.6 What does a 0.8 threshold mean statistically with 3 runs?** Very little. With 3 runs the possible pass rates are 0, 0.333, 0.667, 1.0, so `min_pass_rate=0.8` fails anything but 3/3 (I ran the arithmetic: 0.667 is below, 1.0 is not — exercise 1.6). For a rate estimate, a 95% Wilson interval for 2/3 is very wide [Standard knowledge]. Use more runs, or treat the result as a smoke signal. Source: `.github/workflows/evals.yml L16–21`; `evals/summary.py L30`.

**Q6.7 What would make the system safe for real candidates' data?** Authentication and per-user ownership; encryption in transit and at rest for Redis; a retention policy and redaction for Langfuse prompts; explicit consent and a clear data map for third-party processors (LiveKit, Soniox, Cartesia, Google); authenticated delete (the existing `DELETE /api/candidate-memory` is unauthenticated, and plans/reports have no delete endpoint); stop storing plain email inside the memory value. Source: `routers/memory.py L13`; `memory_store.py L10–23`; `tracing.py L62–66`.

**Q6.8 Can the retry endpoint start two regenerations at once?** `retry_report` reads the status, returns early if `pending`, then later sets `pending`. Those are separate awaits, so two near-simultaneous requests can both pass the `pending` check **[Inference from the code; not reproduced]**; the second would overwrite the same keys (idempotent output, double cost). Fix: an atomic claim (`SET key NX EX`) before starting. The in-process `_retry_tasks` set holds strong references so tasks aren't garbage-collected. Source: `session.py L109–139`; `report_store.py`.

**Q6.9 Why is memory update a lost-update risk, and how would you fix it?** `get_memory` → merge in Python → `save_memory` (full overwrite) with an LLM call (seconds) in between. Two sessions for the same email finishing around the same time can overwrite each other's appended scores/sessions. Fix: optimistic locking with `WATCH/MULTI` or a Lua script, a per-email lock, or an append-only event list with a derived summary. Probability in practice [Not measured] — needs concurrent opted-in sessions on one email. Source: `report_pipeline.py L43–51`; `memory_updater.py L71–131` (C18, D5).

**Q6.10 Trace the failure path if the model returns an invalid category during preparation.** `QuestionCategory(q_data["category"])` raises `ValueError` inside `_generate_questions_once` → propagates out of `retry_async` (attempt 1 of 3, sleeps 1 s) → regenerated (note: the random level sequence is **re-drawn** on each attempt) → after 3 failures the last error is raised → `event_stream`'s generic `except Exception` logs the traceback and emits `stage:"error"` with `GENERIC_PREP_ERROR`; nothing is stored in Redis, so the client must start over. Source: `question_generator.py L171, L411`; `prepare.py L215–223`.

---

# Section A — AI-engineering questions (10)

**QA.1 How do you structure prompts here, and what have you learned about prompt design from the bugs?** Prompts are Python string constants with `{{ }}`-escaped JSON schemas (literal doubled braces appear in prompts that are concatenated rather than `.format`ted — cosmetic, C10), assembled per call. Lessons with evidence: *irrelevant text primes the model* (A1: conditional coding block + negative rule); *a rule in a prompt is a request* (A5); *untrusted data goes in delimiters with an explicit "not instructions" frame* (A7); *the schema in prose still needs code validation*. The interviewer prompt is `INTRO` + the full plan JSON + `DEFAULT_BEHAVIORAL_INSTRUCTIONS`, the last part being the only piece the optimizer may rewrite. Source: `agent/prompts.py L1–71`; `question_generator.py L127, L186`.

**QA.2 How do you get structured output, and how reliable is it?** Request `response_mime_type="application/json"`, strip fences defensively (`_clean_json` takes the first `{…}` block), `json.loads`, then Pydantic or `validate_report`; wrap in `retry_async` (3 tries). Observed failure types: wrong enum value (A1), truncation (A3), dropped tag (A2). No `response_schema` is used. Reliability numbers are [Not measured] beyond the regression scripts' small trials. Source: `resume_analyzer.py L48–66`; `report_generator.py L80–108`.

**QA.3 How do you limit hallucination?** Where the truth is checkable, ground the LLM: code review runs the tests first and the code overwrites `tests_passed/tests_total` with counted results ("ground truth always wins"); the prompt says "Reference actual things the candidate said". Where it isn't, the system doesn't: the report can still invent or mis-score (B3), and the judge's `report_grounded_in_transcript` item is the only check, itself uncalibrated. No citation/quote verification exists; that would be a good improvement (verify each `candidate_answer` is a substring of the transcript). Source: `code_judge.py L70–136`; `report_generator.py L66–68`.

**QA.4 Why is the answer key in the interviewer's prompt, and what's the risk?** The model needs each question's intent and follow-ups, and `ideal_answer_points` come along because the plan JSON is dumped wholesale. Risk: it can leak hints (observed in the original failure: a hint naming specific libraries, and in one stored run the interviewer answered its own questions). Options: send only intent and follow-ups live and keep the key for the report; I would measure leakage with the judge's `leaked_ideal_answer_points` before and after. Source: `prompts.py L63–68`; `evals/judge.py L37`; C8.

**QA.5 How are evals designed, and what are their limits?** Persona-simulated text interviews through the real prompt/tools mirror, deterministic transcript checks, an LLM-judge rubric, pass-rate aggregation with a threshold, and an optimizer that scores failures. Limits: one fixture, uncalibrated judge, same-family judging, tiny n, text only, hand-mirrored tools. See [09](09-testing-evaluation-and-observability.md) §9.5.

**QA.6 What are your cost and latency drivers?** Reasoned, not measured: the full plan JSON (with answer keys and 2–3 follow-ups per question) is resent as the system prompt on every LLM turn; the report call copies every answer into its output (stored reports 29–72 k characters, C6); coding plans use dynamic thinking; retries up to 3×; Langfuse stores prompts. Latency: the voice path's endpointing wait, STT finalisation, LLM TTFT (plus a second call after any tool call), TTS TTFA; none recorded. Token usage *is* recorded per generation in Langfuse when enabled (`usage_details`). Source: `tracing.py L17–29`; [08](08-performance-reliability-and-scaling.md).

**QA.7 Why are the temperatures what they are?** As configured: extraction 0.1 (resume, JD); report and code review 0.2; judge 0.0; memory notes 0.3; question generation 0.4; the live interviewer 0.7; the candidate simulator 0.8 (and 0.4 for writing code). The repo records no tuning study for them [Not recorded]; the pattern is "lower for extraction/evaluation, higher for conversation". Source: grep of `temperature=` in `backend/`, `agent/agent.py L209`, `evals/`.

**QA.8 Why no RAG or embeddings, and when would you add them?** The task is bounded: one resume, one JD, a generated plan — everything fits in context, and a negative search found no vector store or embeddings code. Add retrieval if you want a question bank, company-specific rubrics, or long-term memory across many sessions beyond three weak categories and a 3-sentence note. Source: [02b](02b-ai-and-llm-deep-dive.md) (negative search); `memory_updater.py`.

**QA.9 How do you version and test prompts?** There is no prompt registry. Prompts are in code (git history is the version history). The optimizer saves candidate texts under `evals/optimizer/versions/` and can overwrite `DEFAULT_BEHAVIORAL_INSTRUCTIONS` in `agent/prompts.py` after a regression pass. Tests assert prompt *assembly* only in one case (coding block present/absent); no snapshot test of the interviewer prompt. Improvement: a prompt-assembly snapshot test and evals in CI on a schedule. Source: `evals/optimizer/optimizer.py L119–135`; `test_question_generator_coding_category_bug.py`.

**QA.10 How would you add guardrails for safety and abuse?** Input: size caps exist; add content classification for injection/PII; output: schema validation exists; add checks for profanity or discriminatory language in a hiring-adjacent report; process: rate limits and spend caps; governance: the "hiring_recommendation" in a mock-interview product is an ethics-sensitive output — say it is practice feedback, not a hiring decision, and evaluate bias with varied resumes (the harness has one fixture — a gap). Source: `report_generator.py L8, L43`; C17.

---

# Section B — Voice and real-time questions (8)

**QV.1 Describe one audio turn from mic to speaker.** Browser publishes mic audio to LiveKit; the agent subscribes. Silero VAD marks speech; Soniox streams text (only `is_final` results are recorded); the end-of-turn logic commits the turn; our hook adds the follow-up note; Gemini replies and/or calls a tool (then a second completion); Cartesia streams audio; the agent publishes the track; the browser plays it. **[Library source]** for defaults; per-stage latency [Not measured]. Source: `agent.py L200–243`; [02c](02c-real-time-voice-and-livekit.md) §2c.3.

**QV.2 What are barge-in and false interruptions?** **[Library source]** if the user speaks over the agent for ≥ 0.5 s the agent's speech is cancelled; if the user is then silent for 2 s the interruption is deemed false and the agent resumes. The opening greeting is spoken with `allow_interruptions=False`, and while an utterance is uninterruptible, user audio may be discarded by default — so speech during the greeting can be lost. Source: `interviewer.py L65`; [02c](02c-real-time-voice-and-livekit.md) §2c.8.

**QV.3 How do live-coding events travel between browser and agent?** LiveKit data messages with topics: `coding_challenge` (agent → browser, reliable, payload `{type:"show_editor", question_id, language, problem_statement, starter_code}`) published deterministically from `move_to_next_question`; `code_submission` (browser → agent, `{question_id, code}`). There are no acknowledgements or application-level retries; a failed publish is logged and swallowed. There is no size limit on a submission in the code. Source: `interviewer.py L20–41`; `agent.py L130–163, L246–249`; [02c](02c-real-time-voice-and-livekit.md) §2c.10.

**QV.4 What happens when the candidate disconnects?** `delete_room_on_close=True`; the session closes and shutdown callbacks run: the report callback runs only if `ended_at` is unset, then the trace flush. A client disconnect before the interviewer calls `end_interview` still produces a report from whatever transcript exists. If `end_interview` already set `ended_at`, the shutdown callback skips and relies on the already-spawned task (see Q4.7). Source: `agent.py L251–262, L267–272`.

**QV.5 Why might the interviewer sound slow, and how would you find out?** Candidates (all reasoned): endpointing wait (≈0.55 s normally, up to ~3 s if "unfinished"), STT finalisation, LLM first token — twice if a tool is called — and TTS first audio; plus the hook discarding the preemptive reply. No per-turn metric exists. Plan: log end-of-speech → first-audio per turn, chart p50/p95 in Langfuse/OTel, compare with/without the note. Source: [02c](02c-real-time-voice-and-livekit.md) §2c.12.

**QV.6 What does `AgentSession(userdata=state)` give the tools?** The same `AgentSessionState` instance reaches the `@function_tool` methods as `context.userdata`, so tools mutate shared interview state (advance question, set `ended_at`). The hook uses `self._state` directly. In the eval harness the tools are closures over `state` instead. Source: `agent.py L211`; `interviewer.py L88, L146`; `evals/interview_tools.py L15`.

**QV.7 What are the STT/TTS failure modes?** Jargon misrecognised (Soniox `context` terms unused), only final segments recorded (a long answer becomes several entries), TTS voice fixed, vendor outages unhandled specially (the session just degrades). Turn-detector model missing → coarser turn-taking. None of these paths has a test. Source: `agent.py L201–208, L119–127`.

**QV.8 How would you test the voice path?** Today: nothing automated. Options: a scripted-audio participant joining a test room (needs live providers and costs money), the LiveKit console mode for manual runs (`agent.py console`, needs keys; not run here), recorded-audio fixtures through STT, and latency logging. Be honest that the eval harness covers text behaviour only. Source: [02c](02c-real-time-voice-and-livekit.md) §2c.14; [09](09-testing-evaluation-and-observability.md) §9.5.11.

---

# Section C — Behavioural / STAR questions (framed so you only claim what you did)

**How to use this section.** The *Situation/Task/Result* material below is what the repository can support about the **project**. The *Action* is yours: replace `[your actual contribution]` with what you personally did, or reframe the answer around what you observed and learned. If you did not personally do the thing, say "the team/project did X; I studied it by Y". Never state outcomes the repo does not show (no user counts, no latency gains, no cost savings).

**QB.1 Tell me about a hard bug you debugged.** Situation (supported): question generation failed intermittently, with a validation error on `category="coding"` (A1). Task: make preparation reliable. Action: `[your actual contribution — e.g., reproduced by hand, found the prompt priming, wrote the regression script; delete anything untrue]`. Result (supported): conditional prompt block + explicit negative rule; the commit says the fix was verified with repeated real calls. Do **not** claim a measured failure rate unless you measured it (the commit message says "roughly every other call"; attribute it to the commit).

**QB.2 Tell me about a time you used data to make a technical decision.** Situation: coding plans lost coding-tagged slots with thinking disabled (A2). Task: reliable slot compliance. Action: `[your actual contribution]`. Result (supported): dynamic thinking for coding plans only; the repo states "13/13 perfect" (commit) and "0 misses across 6 trials" (code comment) — quote one, say which, and say you did not re-run it if you didn't.

**QB.3 Tell me about quality or testing work you are proud of.** Situation: LLM behaviour couldn't be checked by unit tests. Task: measure interviewer behaviour. Action: `[your actual contribution — e.g., wrote personas, the checks, the CI workflow, or "I studied and extended X"]`. Result (supported): the harness measured a follow-up cap violation, drove a code-level counter and a deterministic check; stored runs also show silent turns and a rubric contradiction that were later aligned. Don't say "improved pass rate from X to Y" — no before/after rates were recorded.

**QB.4 Tell me about a security concern you raised or handled.** Situation: stored `interviewer_notes` were fed to the next session's prompt (A7). Action: `[your actual contribution]`. Result: untrusted-data framing + a test; the commit itself says it is not a guarantee — say that too.

**QB.5 Tell me about a time you made a mistake or something broke.** Supported options: a fresh checkout of `main` was broken by uncommitted files (A6); an eval-harness bug discarded a generated report (A4). Action: `[your actual contribution]`. Lesson (supported): verify from a clean checkout; make harnesses crash-safe. Only use this if it genuinely happened to you.

**QB.6 Tell me about a time you worked with ambiguity or incomplete information.** Honest, supported angle: the silent-turn finding — measured in evals, root cause unknown — and how you would decide whether to invest. Action: `[your actual contribution or "what I would do"]`. Result: "measured, not fixed". An honest "I don't know yet" with a plan is a legitimate answer.

**QB.7 How did you decide what to build first / how did you prioritise?** Do not invent a process. If you drove it, describe your reasoning `[your actual contribution]`. If not, describe how you'd prioritise this backlog: auth, durable reports, derived scores, voice metrics ([10](10-design-decisions-and-tradeoffs.md) §10.5) with reasons.

**QB.8 Tell me about collaborating, or feedback you received.** The repo shows one Git identity and mentions a "fresh-context review" finding the memory prompt-injection issue (A7) and the broken checkout (A6). If you were the author who received review feedback, `[your actual contribution]`; if you used AI assistance or review, say so plainly. Don't claim teammates that the repo doesn't show.

**Honesty probes you should be ready for.**
- "Which parts did you write yourself?" — answer from your own memory of the work; the repo cannot do this for you.
- "Did you use AI tools?" — answer truthfully. It is fine to say you used assistants and that you can still explain and modify every part; only say it if true and then be ready to modify code live.
- "Can you change X right now?" — practise exercises 1–2 in [12](12-practical-exercises.md) so the answer is yes.

---

# Section D — "What would you improve?" (8)
Each: problem → evidence → fix → how I'd verify. Sort by what you can defend.

1. **Auth and ownership (C3).** Anyone with a session id can fetch tokens/reports; `/api/prepare` is an open spend endpoint. Fix: accounts or signed session links, rate limits, spend caps. Verify: an unauthenticated request returns 401; a load test of prepare shows throttling.
2. **Durable, backend-side report generation (D1, D3, C4).** Move generation out of the shutdown path (worker triggered by last autosave + explicit end), store `started_at`, expire stale `pending`, persist to a database. Verify: kill the agent mid-interview; a report still appears; report-success-rate metric.
3. **Derive scores in code (B3, C7).** Compute overall/category from per-question scores or validate consistency and retry. Verify: a test over stored reports; judge `report_grounded_in_transcript` rate.
4. **Voice metrics (C17, §2c.12).** Per-turn latency, silent-turn counter, report success; dashboards and alerts. Verify: p50/p95 visible; before/after of the hook.
5. **Hard follow-up guarantee (C2) — only if measured violations justify the naturalness cost.** Verify: the cap check over ≥ 20 `vague` runs.
6. **Calibrate and broaden evals (C17).** More fixtures (seniority, domains), human-labelled judge calibration, scheduled evals with more runs, fix the optimizer's "error baseline = −1" flaw. Verify: κ, wider intervals.
7. **Retry hygiene (D2).** Classify errors, add jitter and a breaker; atomic retry claim (Q6.8). Verify: simulated outage test (exercise 2.3).
8. **Privacy (C15).** Redaction/retention in Langfuse, no plain email in memory values, authenticated delete, consent text. Verify: grep traces for resume content; deletion test.

---

# Section E — Questions to ask the interviewer (10)
Pick 3–4; tailor to their stage and stack. None assumes facts about them.
1. How do you evaluate LLM-powered features today — offline evals, human review, production monitoring — and who owns them?
2. What does your path from a prototype agent to a production service look like (auth, data retention, SLOs)?
3. How do you handle prompt and model changes — versioning, regression evals, rollbacks?
4. Which parts of the stack are real-time or latency-critical, and how do you measure it?
5. What does on-call look like for the AI systems, and what was the last notable incident and what changed afterward?
6. How is data privacy handled for user content sent to model vendors?
7. How do engineers here decide between prompt changes, code-level enforcement and fine-tuning?
8. What would a successful first 90 days look like for this role, and what would I own?
9. How does the team balance building features and paying down reliability/test debt?
10. What do you wish candidates asked more about?

---

# Scenario questions
- *"Reports are intermittently missing after a call."* See Q4.7; plan: status key states, logs, a success-rate metric; fix: move generation and add staleness handling.
- *"Latency jumped after a prompt change."* Compare prompt sizes (the full plan is resent each turn); check whether the hook is invalidating preemptive generation; use Langfuse for backend calls; the voice path has no metrics, so add them first.
- *"Add a new interview type (e.g., system design)."* Touch `INTERVIEW_TYPE_PRESETS`/`resolve_plan_options` (`prepare.py L25–52`), the prompt blocks, the frontend selector and TS types (drift risk, C12), add a test and an eval persona.
- *"A candidate says the interviewer revealed an answer."* Check the transcript; compare with `ideal_answer_points`; run `vague` evals with the leak rubric; consider removing the key from the live prompt.
- *"Gemini is down for ten minutes."* All three retries fail → generic prepare error; live interviews: each turn's LLM call fails; the report goes `failed` and can be retried; no queue, no fallback model.

## Misconceptions checklist
- "Reconnect resumes the interview" — it restarts (C1).
- "The follow-up cap is enforced" — counted and requested, not forced.
- "Difficulty is guaranteed" — random per slot; compliance logged only.
- "Backend calls Piston" — only the agent (and the eval driver).
- "There's RAG/embeddings" — none.
- "Evals run in CI" — manual workflow only.
- "Tests prove the voice pipeline" — they don't touch it.
- "All 53 tests run everywhere" — the follow-up-cap file is skipped without `livekit.agents`; the smoke test skips without Redis.
- "Langfuse is mandatory / absent library is fine" — absent keys are fine; an absent library is an import error.

---

# Coverage and readiness (the honest answer)

**What this bank covers well:** the architecture and every main flow in this repo; specific bugs and the evidence for them; the testing/eval story (verified by running things); security/privacy gaps; real-time voice concepts at the level the code uses them; a defensible improvement list; question-asking and STAR scaffolding.

**What it does not cover — you must prepare these separately:**
- **Your own contribution.** The `[your actual contribution]` blanks are the most important part of the interview. Write them out and rehearse them.
- **Generic technical screens:** data structures and algorithms, live coding in an unfamiliar problem, Python/TypeScript language depth, SQL, networking basics beyond what's here, classic system design (this repo does not contain a database design, a queue, a cache invalidation problem, etc.).
- **ML fundamentals** (training, fine-tuning, embeddings math, evaluation statistics beyond what's mentioned), and general LLM-ops patterns not used here (RAG pipelines, vector DBs, fine-tuning, guardrail frameworks).
- **Frontend depth:** only touched at the level the integration needs ([02d](02d-frontend-technologies.md)); there are no frontend tests.
- **Anything the repo doesn't measure:** latency, cost, load, user outcomes. Honest "not measured, here's how I'd measure" is the answer; this bank cannot supply numbers.
- **Company-specific preparation** (their product, stack, questions).

**Suggested use (a plan, not a guarantee):** (1) pick two flows and give both 2-minute walkthroughs aloud without notes; (2) do exercises 1.1–1.7 and 2.1–2.7 in [12](12-practical-exercises.md) so you can modify code on demand; (3) write and rehearse your real STAR answers; (4) rehearse the "what I would improve" list; (5) drill the six levels by covering the answer and speaking first. If you can do (1)–(3) from memory, you are well prepared on this project; the general screens above remain separate work.
