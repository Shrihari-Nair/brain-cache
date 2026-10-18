# 07 — Interview Preparation

This chapter turns the project into answers you can say aloud. It is organised by topic, and the questions most likely to come up are marked **(High value)**. Each entry has a concise answer you can give in about a minute, a deeper explanation, a concrete example from this project, likely follow-ups, and nuances or common misconceptions. It links to chapters [02](02-architecture.md) to [06](06-quality-and-reliability.md) instead of re-explaining them.

Three ground rules protect you:
1. **Only claim what you did.** Behavioural prompts use `[your actual contribution]`. The repository has a single Git identity and does not say who did what, so it cannot supply your story. Fill the blanks with true things or reframe as "the project did X; I studied it by Y".
2. **Never quote unmeasured numbers.** The repo has no latency, cost, load, accuracy or user numbers. "Not measured; here is how I would measure it" is a strong answer.
3. **"I haven't root-caused that; here is how I would trace it" beats an invented cause.**

---

## 0. Model walkthroughs

### Two-minute walkthrough: the architecture
"MockMind is an AI mock-interview platform. A candidate uploads a resume and a job description, and then has a spoken interview that follows a personalised plan, ending in a scored report.

There are four things we run and a few we rent. The **frontend** is a Next.js app with a setup page, an interview page and a report page. The **backend** is FastAPI with three jobs: prepare an interview, hand out a token to join the voice room, and serve the report. The **agent** is a LiveKit worker, one job per interview room, that does speech-to-text, calls Gemini, and speaks back with text-to-speech. **Redis** holds all shared state. A **Piston** sandbox runs candidate code, and optional **Langfuse** records traces.

The key design point is that the backend and the agent never call each other. They share a contract through Redis: the backend writes the plan, the agent reads it, the agent writes the transcript and report status, the backend serves them. That decouples a web service from a long-lived real-time process.

The second point is how much autonomy the AI has. Preparation is a fixed workflow of single model calls: analyse the resume, analyse the job description, generate questions, with difficulty assigned by code. The live interviewer is a tool-calling model, but it works inside a question sequence the code controls. So it is a constrained agent, not a free-running one. And a prompt rule is a request, not enforcement: the follow-up limit is counted in code and measured by a check, but still soft.

What I would flag first: no authentication, Redis only has default snapshot persistence, the report is generated in the agent's shutdown path, and nothing about latency or load is measured." (See [02](02-architecture.md).)

### Two-minute walkthrough: one flow, end to end
"A candidate uploads a PDF and a job description. The browser posts to the backend and gets a server-sent-event stream back, so it can show progress. The backend validates size and type, extracts text, then runs resume analysis and job-description analysis, each a Gemini call that must return JSON, validated and retried up to three times with short backoff. Then question generation: code first picks a difficulty for each slot, and the model fills in the questions. The plan is stored in Redis with a four-hour expiry and the stream ends with a session id.

The browser requests a token. The backend checks the plan exists and signs a short-lived token that lets this browser join a room named for the session, with an explicit instruction to dispatch our agent and the session id as metadata. LiveKit starts an agent job, which loads the plan from Redis, builds the voice pipeline and speaks the opening line and first question.

Each candidate turn passes a hook that counts follow-ups and adds a note telling the model where it stands. When it is done with a question, the model calls a tool to advance; for a coding question the code, not the model, opens the editor, runs the submission in the sandbox, and the model reviews it against real test results.

At the end, or at shutdown, the agent saves the transcript, sets the report status to pending, makes one Gemini call, validates the result, and stores it as ready or failed. The report page polls; failure shows a retry button." (See [03](03-how-it-works.md).)

---

## 1. Overview questions

### 1.1 Describe the project and your role. **(High value)**
**Concise answer.** An AI mock-interview platform: resume and job description in, a personalised spoken interview, a structured report out. `[your actual contribution: one sentence]`.

**Deeper.** Lead with the user problem, then the shape (prepare, interview, report), then one standout design idea such as the constrained agent or the eval harness. End with your part.

**Project example.** "Preparation is a pipeline of model calls; the interview is a voice agent following a plan; the report is one validated model call with a retry path."

**Follow-ups.** Who uses it? (Do not invent users; the repo has none recorded.) How big was the team? (Answer truthfully; the repo shows one Git identity.) What would you build next?

**Nuances.** Do not oversell it as production: it is a working prototype with real engineering around it and known gaps.

### 1.2 What makes this "agentic"? Is it really an agent? **(High value)**
**Concise answer.** Only the live interviewer is agent-like: a model that decides when to call tools (advance, end, look up the current question) inside a question order the code controls. Preparation and reporting are workflows of single model calls.

**Deeper.** A workflow has a fixed, code-defined sequence; an agent decides its own next step. Constraining the agent is a deliberate reliability choice: the model gets conversational freedom where it helps and no freedom where mistakes are costly (what the question list is, when the interview ends, opening the editor). See [04](04-core-concepts.md).

**Project example.** The editor opens when the code advances to a coding question, not because the model remembered to ask for it.

**Follow-ups.** Why not let it plan the whole interview? Is there multi-agent or retrieval? (No: no multi-agent, embeddings or RAG.)

**Nuances.** "Agentic" is a spectrum; say where on it you sit and why.

---

## 2. Fundamentals

### 2.1 What is server-sent events, and why use it for preparation?
**Concise answer.** One-way streaming over a normal HTTP response; the server pushes events as they happen. Used so the user sees progress during three slow model calls.

**Deeper.** Alternatives: polling with a job id (survives reconnects, more code), WebSockets (two-way, overkill). The cost of SSE here is that if the connection drops, the work is not resumable.

**Project example.** Stages stream out, then a final `complete` with the session id, or a friendly `error`.

**Follow-ups.** What if the client disconnects? What about a heartbeat during a long retry?

**Nuances.** SSE is a UX choice, not a speed-up.

### 2.2 Why does the LLM output need validation? **(High value)**
**Concise answer.** A model produces text that looks like what you asked for but can be malformed, truncated, or semantically wrong. Code must treat it as untrusted input.

**Deeper.** Ask for JSON, parse it, validate structure (Pydantic or a validator), retry on failure. Structural validation is not semantic validation: a number can be well-formed and still wrong.

**Project example.** The report validator rejects missing keys and unknown recommendations, but a score of 99 would pass, and overall and per-question scores were observed to disagree in stored runs. See [04](04-core-concepts.md).

**Follow-ups.** Why not native schema-constrained output? What does a retry cost?

**Nuances.** Native structured output reduces parse failures but does not remove the need for semantic checks.

### 2.3 Why do thinking tokens matter?
**Concise answer.** Hidden reasoning tokens spend from the same output budget as the visible answer, so too much thinking can truncate the answer.

**Deeper.** That is why thinking is off for extraction and reporting and on only where per-slot constraints were being dropped.

**Project example.** The first real eval run crashed on truncated report JSON; the fix was thinking off, a larger output ceiling, JSON mime type, plus validation. Coding-plan generation uses dynamic thinking because coding tags were being dropped without it.

**Follow-ups.** How did you decide per task? How would you detect truncation in code? (Check the finish reason.)

**Nuances.** The recorded "13 of 13" and "0 misses in 6" figures are repository claims; we did not re-measure.

### 2.4 Why Redis with TTLs?
**Concise answer.** Fast, simple key-value storage shared by two processes, where expiry gives automatic cleanup and a privacy-friendly default.

**Deeper.** The cost is durability and queryability.

**Project example.** Plans and session state live four hours, reports a day, memory a year.

**Follow-ups.** Is data lost on restart?

**Nuances.** Misconception: "no persistence". Compose uses default snapshots with AOF off and an anonymous volume: writes since the last snapshot can be lost and the volume may not survive being recreated.

### 2.4b Explain real-time voice basics in a sentence each.
VAD detects speech; STT turns audio to text; endpointing decides when the speaker has finished; the LLM replies; TTS speaks; interruption handling cancels speech when the user barges in. See [01-foundations](01-foundations.md) and [04](04-core-concepts.md).

---

## 3. Architecture

### 3.1 Walk me through the architecture. **(High value)**
Use the walkthrough in section 0, then be ready to go one level down on any box.

**Follow-ups.** Where is the trust boundary? Which component fails worst? Which parts are stateless?

**Nuances.** Say "the backend and agent never call each other" early; it is the clearest sign you understand it.

### 3.2 How do the components communicate? **(High value)**
**Concise answer.** Browser to backend over HTTP (SSE for preparation); browser to LiveKit over WebRTC, with a signed token from the backend; backend and agent only through Redis; agent to Piston over HTTP; browser to agent for coding events over the LiveKit data channel.

**Deeper.** The Redis contract is key names plus shared data models. There is no direct RPC, so each side can fail or restart without synchronous coupling.

**Project example.** Report status is a small state machine in Redis (`pending`, `ready`, `failed`) that the backend reads while the agent writes it.

**Follow-ups.** What breaks if the key schema changes? (Both sides must change together; no versioning.) What replaces this at scale? (A queue or database-backed job.)

**Nuances.** Piston is called only by the agent (and the eval driver), not the backend.

### 3.3 Why is the agent separate from the backend?
**Concise answer.** The agent is a long-lived real-time worker with a different lifecycle, scaling profile and dependencies than a request/response API.

**Deeper.** One job process per live interview versus short HTTP handlers. Separation lets each scale and fail independently. Cost: duplicated logic and a Redis-shaped contract to keep consistent.

**Follow-ups.** Why not run the agent inside the backend? (Blocking, resource isolation, framework model.)

### 3.4 Why explicit agent dispatch with a token?
**Concise answer.** The token tells LiveKit which room the browser may join and which agent to send, carrying the session id so the agent can load the right plan.

**Nuances.** The token is minted locally with no network call; the plan check happens first so an expired session fails fast with a 404.

---

## 4. Workflows

### 4.1 Explain the main feature end to end. **(High value)**
Use the flow walkthrough in section 0. Then pick a failure path to discuss.

### 4.2 What happens when the candidate "reconnects"?
**Concise answer.** It restarts the interview. The page reloads and the agent loads only the plan, not the saved transcript.

**Deeper.** Autosave exists to survive an agent crash, but resume is not implemented.

**Nuances.** Common misconception: reconnect resumes. It does not. Good example of knowing the difference between a feature that exists and a design that was intended.

### 4.3 How does a coding question work?
**Concise answer.** The code advances to the coding question, sends the problem and starter code to the browser over a data channel, receives the submission, executes it against tests in a sandbox, then has the model review with real test results as ground truth.

**Deeper.** The counted pass and fail numbers overwrite whatever the model claims. If the sandbox is down, tests simply fail and the review still happens, so infrastructure failure looks like wrong code.

**Follow-ups.** How do you stop infinite loops? (Per-test timeout.) Why not trust the model to judge correctness? (Hallucination; ground it.)

### 4.4 How is the report produced and retrieved?
**Concise answer.** One model call over the transcript, validated, stored with a status; the report page reads status and shows result, failure with retry, or pending.

**Nuances.** Report generation lives in the agent's end-of-session path, which has no HTTP caller to report errors to; hence the status key and retry endpoint. `pending` has no staleness timeout.

---

## 5. Design decisions

### 5.1 Why this technology stack? **(High value)**
**Concise answer.** Pick by role, give the reason, then the trade-off. FastAPI: async, typed, SSE support. Redis: simple shared ephemeral state. LiveKit Agents: handles the hard real-time pieces (WebRTC, VAD, turn-taking, interruptions) so we write interview logic. Gemini Flash: one fast model for every call. Piston: isolation for untrusted code. Pydantic: validate model output.

**Deeper.** Be honest: the repository does not record why Gemini Flash over alternatives, or most vendor choices. Say what the choice buys and what you would test before changing it.

**Follow-ups.** What is the lock-in? (Single model vendor, no fallback; vendors are swappable in a cascaded pipeline.) Why not a speech-to-speech model? (Cascaded gives text at every stage for tools, transcripts and evals, at the price of more hops of latency.)

**Nuances.** "Why X" answers that only praise X sound junior; always add a cost.

### 5.2 Prompt rule versus code enforcement. **(High value)**
**Concise answer.** Prompts are free and natural but unreliable; code is reliable but rigid. This project mixes: code counts and tells the model, a check measures, nothing forces.

**Project example.** The follow-up cap of two. Origin: an eval found three follow-ups plus a hint naming specific libraries. Now: a counter, a per-turn note stating the standing and, at the limit, an explicit stop instruction, plus a deterministic eval check. It remains soft: the model can ignore the note, nothing auto-advances.

**Follow-ups.** How would you make it hard? (Code-driven advance or an output validator, at the cost of naturalness and latency.) Is a hard guarantee worth it? (Only if measured violations justify it.)

**Nuances.** The per-turn note changes the conversation context, which may discard the framework's speculative reply and add latency. Magnitude not measured.

### 5.3 Why is difficulty assigned in code?
**Concise answer.** Models are poor at hitting a distribution, so code draws the difficulty per slot and the model fills the question.

**Nuances.** It is a weighted random draw, not a guarantee; compliance is only logged. The README's word "guarantee" overstates it.

### 5.4 Redis versus a database.
**Concise answer.** Redis is right for ephemeral plans and sessions; durable reports and memory belong in a database.

**Project example.** Memory updates are read-modify-write with an LLM call in between, so two concurrent sessions for one email can overwrite each other. Fix: optimistic locking, a per-email lock or an append-only log. Probability in practice not measured.

### 5.5 Why SSE for prepare, why generate the report in the agent?
See 2.1 and 4.4: simplicity now, resumability and sturdiness later.

---

## 6. Performance and scalability

### 6.1 Latency has increased. How do you investigate? **(High value)**
**Concise answer.** First decide which path: preparation, voice turn or report. Break the path into stages, find which stage changed, then test the suspected cause.

**Deeper.** Method:
1. Reproduce and name the path.
2. Look at per-stage timings. For backend model calls, traced durations and token counts show whether input size, thinking or provider latency changed. For voice, a turn is end-of-speech wait plus speech-to-text finalisation plus model time-to-first-token (doubled when a tool is called) plus text-to-speech first audio.
3. Check what changed: prompt size (the whole plan, including answer keys, is resent on every turn), thinking settings, model, retries, the per-turn note.
4. Use p50 and p95, not averages.
5. Fix, then confirm with the same measurement.

**Project example.** Honest gap: no voice-turn latency metric exists, so step 2 needs instrumentation first. That answer ("I would add it before guessing") is a good one.

**Follow-ups.** What would you instrument? (Per-turn timestamps and the framework's metrics events.) What if only some users are slow? (Look at p95 and per-session traces.)

**Nuances.** Retries multiply latency: three attempts plus short waits. No per-call timeout means a hung request can wait very long.

### 6.2 How would this scale to many concurrent interviews? **(High value)**
**Concise answer.** The backend is largely stateless and scales horizontally. The real limits are model quota, agent capacity (one process per live interview), Redis durability, and sandbox throughput.

**Deeper.** Likely order of bottlenecks (reasoned, not measured): model quota and latency; agent CPU and memory; Redis durability before throughput; event-loop blocking from resume parsing; sandbox calls executed one test at a time. Improvements: a concurrency limiter with jitter and a circuit breaker, a queue plus worker for reports, a database for durable data, auth and rate limits, more agent workers, and metrics first.

**Project example.** We measured only two small things (voice-activity detection CPU on synthetic audio, and a lower-bound memory for an agent process); there is no end-to-end capacity figure.

**Follow-ups.** First number you would get? (The model quota.) What is the cost driver? (Speech minutes, text-to-speech characters, model input tokens.)

**Nuances.** Do not claim a capacity. Say what you would measure.

### 6.3 What does retry with backoff do and where does it fall short?
**Concise answer.** It recovers from brief failures by waiting one then two seconds, up to three attempts.

**Deeper.** Shortfalls: every exception is retried (even a missing key), no jitter so clients retry in lockstep, no circuit breaker, no deadline; each retry of a bad response is a full paid call.

**Project example.** Stored eval runs show provider 503s outlasting the roughly three seconds of waiting.

---

## 7. Failures and debugging

### 7.1 What happens on invalid input or a bad model response? **(High value)**
**Concise answer.** Bad uploads (type, size, empty, overlong text) are rejected with friendly messages before any model call. Bad model output is validated and retried; after three failures the user gets a generic error and nothing is stored.

**Deeper.** The test suite asserts that an unexpected exception shows a generic message and never leaks internals. Because nothing is stored on failure, the client starts over.

**Follow-ups.** What if the model returns an invalid enum? (Raise, retry; the random difficulty sequence is redrawn on each attempt.) What if it returns fewer questions than asked? (Not checked; the plan can be silently shorter than promised. Observed in stored runs.)

**Nuances.** Distinguish a confirmed historical issue (the model confusing `coding` with category, fixed by conditional prompt text and an explicit rule) from a current limitation (no count check).

### 7.2 An external dependency is down. What happens? **(High value)**
**Concise answer.** Depends which one; most failures degrade rather than crash, but few have a good user message.

| Dependency | Effect |
|---|---|
| Gemini | Prepare: three retries, then a generic error. Live turns: framework retries then closes the session after repeated errors. Report: marked failed, retry button. No fallback model, no queue |
| Redis | Down at start: backend refuses to start. Down later: unhandled errors, except the memory path. Agent cannot load a plan, so the job fails |
| Sandbox (Piston) | Tests recorded as failed, so outage looks like wrong code |
| Speech vendors | Framework retries, then closes the session; the candidate sees the agent stop hearing or speaking |
| LiveKit | Token is minted locally so the outage shows at browser connect; agent not dispatched gives a "taking too long" overlay |

**Nuances.** Say which of these are verified in code versus reasoned from library behaviour. Full detail in [05](05-engineering-decisions.md).

### 7.3 A user says the report never arrived. **(High value)**
**Concise answer.** Check the status key. Pending or unknown returns 404, failed returns 502, ready returns 200. Then the pipeline log, then whether the transcript was saved, then use the retry endpoint.

**Deeper.** Plausible causes: the agent died before generating, the model call failed, the key expired (reports last a day), or the status is stuck pending because there is no staleness timeout. Cause frequency is not measured.

**Follow-ups.** Can the retry endpoint start two regenerations? (Yes, plausibly: the check and set are separate awaits. Inferred, not reproduced. Fix: an atomic claim.)

### 7.4 What bugs have you hit, and how were they found? **(High value, behavioural)**
Pick from confirmed history, and attribute nothing to yourself unless true.
- Model confusing question type with category crashed preparation intermittently (fixed with conditional prompt blocks and a negative rule).
- Coding tags dropped when thinking was off (fixed with dynamic thinking for coding plans).
- Report JSON truncated by thinking tokens (found by the first real eval run).
- An eval-harness bug discarded a generated report on a later failure.
- A fresh checkout of main was broken by uncommitted files (found in a final review).
- A two-hop prompt-injection path through stored memory notes, mitigated by framing them as untrusted data.

**Behavioural prompt.** "Tell me about a hard bug." Situation and result come from the repo; the action is `[your actual contribution]`. Do not state failure rates the repo never measured.

---

## 8. Testing and evaluation

### 8.1 How do you know it works? **(High value)**
**Concise answer.** Three layers: fast tests with the model stubbed prove our code; a persona-driven eval harness samples the real model's behaviour; tracing explains individual sessions. And I can tell you what none of them cover.

**Deeper.** See [06](06-quality-and-reliability.md). The harness uses a simulated candidate, the real preparation and report code, and a text version of the interview loop; it scores transcripts with deterministic checks plus an LLM judge, and the deterministic check overrides the judge.

**Project example.** The harness measured a follow-up cap violation, which led to a counter, a note and a check. It also revealed a rubric contradicting the prompt.

**Follow-ups.** How reliable is the judge? (Not calibrated; same model family; I would label a sample by hand and compute agreement.) Is it in CI? (No: manual workflow, because it costs money and is nondeterministic.) What does a 0.8 threshold mean with three runs? (Only three of three passes it.)

**Nuances.** "Tests prove code, evals measure models" is a good line. Misconceptions: evals run automatically; tests cover the voice path.

### 8.2 What is tested offline, and what is not?
Covered: retry, validation, report pipeline and endpoints, upload limits and error handling, the follow-up counter, prompt assembly for one case, and the harness's own logic, plus one Redis-backed smoke test across both services. Not covered: frontend, voice, the agent runtime, sandbox integration, load, and the model itself.

### 8.3 How would you test the voice path?
Options: a scripted-audio participant in a test room (needs live providers and costs money), recorded audio through speech-to-text, the framework's console mode for manual runs, and latency logging. The existing harness is text-only and says nothing about speech accuracy, interruptions or timing.

### 8.4 What is LLM-as-judge and what are its risks?
A model scoring another model's output against a rubric. Risks: bias, self-preference when the same family judges, rubric contradictions (observed here), polarity mistakes (one field is "true means bad"), and no calibration against humans. Mitigate with deterministic checks, human-labelled samples and agreement statistics.

### 8.5 Can you trust a single eval pass?
No. Pass rates need many runs; three runs give coarse steps of one third. One passing run of the injection persona shows one phrasing resisted once, not immunity.

---

## 9. Improvements and alternatives

### 9.1 What would you improve first? **(High value)**
Give problem, evidence, fix, how to verify; keep four categories distinct (confirmed issue, design limitation, hypothetical failure, suggestion). Sort by what you can defend.

1. **Auth, ownership, rate limits and spend caps.** Prepare is an open spend endpoint; anyone with a session id can fetch a token or report. Verify: unauthenticated requests get 401; throttling under load.
2. **Durable, backend-side report generation** with a stale-pending timeout and a database for reports. Verify: kill the agent mid-interview and still get a report.
3. **Derive or reconcile scores in code**, since overall and per-question scores disagreed in stored runs. Verify with a test over stored reports.
4. **Voice metrics and alerts** (turn latency p50/p95, silent turns, report success rate).
5. **Broader, calibrated, scheduled evals**: more fixtures, human-labelled judge calibration, more runs.
6. **Retry hygiene**: classify errors, add jitter, a breaker and timeouts.
7. **Privacy**: redaction and retention for traces, no plain email in memory, authenticated delete, consent text.
8. **Remove the answer key from the live prompt** if leakage evidence justifies it; measure with the judge's leakage field before and after.

### 9.2 What alternatives did you consider?
Be careful: the repo rarely records alternatives considered. Say "a reasonable alternative would be X, with these trade-offs", not "we chose Y over X after testing". Examples: native structured output versus parse-and-validate; speech-to-speech versus cascaded; database versus Redis; job queue versus an agent shutdown hook; third-party eval tools versus a custom harness (custom reuses the real production code but has a hand-mirrored tool set that can drift).

### 9.3 How would you handle prompt injection and unsafe content? **(High value)**
Current: size and type limits, structural validation, and framing stored notes as untrusted data with a test. Absent: sanitisation, a classifier, framing in the resume and job-description analysers, spend limits. Evals cover only spoken injection, not resume injection. Frame it as defence in depth, never a guarantee.

---

## 10. Honest limits: what the repo does not measure

Say these confidently; they show maturity.
- End-to-end voice latency and per-stage breakdown.
- Cost per interview.
- Capacity or concurrency limits (only two small local measurements exist).
- Speech recognition accuracy.
- Report success rate in live use.
- Frequency of silent turns in voice (seen in text evals only; cause unknown).
- Judge agreement with humans.
- Pass rates with statistical confidence (tiny n, one resume and job description).
- Anything about real users.
- Effect of the per-turn note on latency.
- Whether the manual eval workflow has ever been run in CI.

---

## 11. Behavioural prompts (placeholders only)

For each: Situation and Result can be drawn from the confirmed history above; **Action is yours.** Do not state outcomes the repo does not show (no user counts, latency gains or cost savings).

1. **A hard bug.** Situation: intermittent preparation failures or truncated reports. Action: `[your actual contribution]`. Result: what changed, with only evidence you have.
2. **A data-driven decision.** Situation: coding tags dropped without thinking. Action: `[your actual contribution]`. Result: say which recorded figure you are quoting and that you did or did not re-run it.
3. **Quality work you are proud of.** Situation: LLM behaviour could not be unit tested. Action: `[your actual contribution: built personas, checks, CI, or studied and extended them]`.
4. **A security concern.** Situation: stored notes re-injected into later prompts. Action: `[your actual contribution]`. Result: framing plus a test, openly not a guarantee.
5. **A mistake.** Options: a broken fresh checkout; an eval bug that discarded a report. Action: `[your actual contribution]`. Lesson: verify from a clean checkout; make harnesses crash-safe.
6. **Ambiguity.** Situation: silent interviewer turns measured but root cause unknown. Action: `[your actual contribution or what you would do]`.
7. **Prioritisation.** `[your actual contribution]`, or describe how you would rank the improvements in section 9.
8. **Collaboration and feedback.** `[your actual contribution]`. If you used AI assistance, say so plainly and be ready to modify the code live.

---

## 12. Questions to ask the interviewer
- How do you evaluate LLM features today: offline evals, human review, production monitoring?
- How do you version and regression-test prompts and model changes?
- Which paths are latency-critical and how do you measure them?
- What was the last notable incident and what changed afterwards?
- How do you decide between prompt changes, code-level enforcement and fine-tuning?

---

## 13. Misconceptions checklist
- Reconnect resumes the interview: it restarts.
- The follow-up cap is enforced: counted and requested, not forced.
- Difficulty is guaranteed: weighted random draw.
- The backend calls Piston: only the agent does.
- There is RAG, embeddings or multi-agent: none.
- Evals run in CI: manual workflow only.
- Tests prove the voice pipeline: they do not touch it.
- Redis has no persistence: it has default snapshots, AOF off.
- Langfuse missing is fine: missing keys are fine; a missing library fails at import.
- All tests run everywhere: the follow-up-cap file needs the LiveKit package and the smoke test needs Redis.

## Readiness

This chapter is necessary but not sufficient. It cannot supply your own contribution, fluency (say answers aloud until you stop reading them), or generic screens such as algorithms, language depth, SQL or classic system design. Practise the two walkthroughs, the failure and latency questions, and the "how do you know it works" answer until each fits in two minutes.

## Recap
- Lead with architecture and the constrained-agent idea; remember backend and agent share only Redis.
- Show you know the difference between a prompt request and code enforcement.
- Answer failure, latency, scaling and testing questions with a method, then state what is unmeasured.
- Use placeholders for your own contribution and invent nothing.
