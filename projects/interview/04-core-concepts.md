# 04 - Core Concepts

This chapter teaches the twelve ideas that make this system work, one concept at a time. For each you get the first-principles intuition, the mechanism, how the project uses it, where it falls short, what else you could have done, and the questions an interviewer is likely to ask. If you can explain these twelve ideas in your own words, you can explain the whole project.

Evidence labels used sparingly: **[Verified]** read in the repo or git, **[Measured]** computed locally without network calls, **[Inference]** reasoned, **[Not measured]**. Prerequisite vocabulary (tokens, JSON mode, tool calling, WebRTC, STT/TTS/VAD) lives in [01-foundations](01-foundations.md); the components themselves are in [02-architecture](02-architecture.md); the end-to-end flows are in [03-how-it-works](03-how-it-works.md). Problems that actually occurred are in [05-engineering-decisions](05-engineering-decisions.md).

**Essential** concepts: 1, 2, 5, 6, 7, 8. **Optional-advanced**: 3, 4, 9, 10, 11, 12 (still worth knowing for deeper interviews).

---

## 1. Structured LLM output and validation

**First principles.** A language model produces text one token at a time. "Return JSON" is a request, not a type system. The model can wrap the JSON in prose, wrap it in markdown fences, use an enum value you did not allow, forget a key, or stop in the middle of a string because it ran out of its output budget. Any code that feeds model output into typed logic must therefore treat the output as untrusted input from a flaky colleague: parse it, validate it, and have a plan when it is wrong.

**Mechanism.** Every backend LLM call in this project follows the same four steps:

1. Ask for JSON (`response_mime_type="application/json"`), which makes the model much more likely to emit bare JSON.
2. Clean the text (strip markdown fences, take from the first `{` to the last `}`).
3. Parse it (`json.loads`), then validate it against a Pydantic model (resume profile, job profile, question plan) or a hand-written check (the report).
4. If any step raises, the whole attempt is retried (concept 2).

```mermaid
flowchart LR
    R["Raw model text"] --> C["Clean fences / prose"]
    C --> P["Parse JSON"]
    P --> V["Validate against schema"]
    V --> OK(["Typed object"])
    P -- "syntax error / truncated" --> RT["Retry whole attempt"]
    V -- "wrong shape or enum" --> RT
    RT --> R
    RT -- "attempts exhausted" --> ERR(["Error surfaced to caller"])
```

The diagram asks one question: where can a bad answer be caught? At parse time (broken syntax, truncation) and at validation time (valid JSON, wrong content); both feed the same retry loop.

**How the project uses it.** Preparation (resume, job description, question plan) and report generation all use this pipeline. The report has a hand-written validator that insists on an overall score, a hiring recommendation from a fixed set, and an object of category scores. **[Verified]**

**Why it helps.** Garbage never reaches the interview. A wrong enum value (for example a question whose category is `coding`, see [05](05-engineering-decisions.md), challenge 1) becomes a retried exception rather than a corrupted plan in Redis.

**Limitations.**
- Validation checks shape, not truth. The report validator does not check numeric ranges, so a category score of 99 passes **[Measured]**; and it cannot notice that the overall score contradicts the per-question scores.
- Nothing checks the plan's size or the number of coding questions after generation, so a plan can come back shorter than requested.
- The fence-stripping helper is copy-pasted into five files and its greedy regex mangles a top-level JSON array **[Measured]**. Harmless today because every schema is an object.
- Provider-enforced schemas (`response_schema`) are available in the SDK and unused; the original design spec said it would use them, and no reason for the change is recorded.

**Alternatives.** Provider-enforced schema output (removes most syntax failures); function calling used as a "return this structure" mechanism; JSON-repair libraries (rescue truncated output but can silently produce wrong data).

**Related.** Retry (2), thinking budget (3), soft vs hard enforcement (5).

**Likely interview questions.**
- "What does JSON mode guarantee?" Syntax-ish likelihood, not schema conformity and not completeness. Say what you do about the rest.
- "Why validate if you asked nicely for JSON?" Because the model is probabilistic; validation turns a silent corruption into a loud, retryable failure.

---

## 2. Retry with backoff around the whole attempt

**First principles.** Calls to a hosted model fail for two different reasons: transient infrastructure trouble (rate limit, 503, timeout) and content trouble (truncated or malformed JSON). Both are often fixed by simply asking again, because sampling is random and outages are often brief. Exponential backoff means waiting a little longer between each attempt so you do not hammer a struggling service.

**Mechanism.** A small helper runs a zero-argument async function up to three times, sleeping 1 s then 2 s between attempts, and re-raises the last exception if all fail. The important design point is *what is inside the retried function*: the model call, the parsing and the validation. So a malformed answer is retried exactly like a network error. **[Verified]** `retry_async` in `backend/services/llm_retry.py`.

**How the project uses it.** Wraps resume analysis, job analysis, question generation and report generation, and (with a 2 s base delay) the eval harness's chat calls. It does **not** wrap code review or the memory-notes call, so those two behave differently under a transient error: code review turns into a spoken apology, notes generation is simply skipped. **[Verified]**

**Why it helps.** One bad sample no longer fails a user's whole preparation. Putting parse and validation inside the retried unit is the neat part: it makes "the model gave nonsense" a first-class retryable event.

**Limitations.**
- It retries every exception, including programmer errors such as a missing API key, wasting three attempts.
- No jitter, no `Retry-After` handling, no circuit breaker, no total deadline, and no per-call timeout anywhere in the app code. **[Verified]**
- The window is about 3 seconds of waiting. Stored eval runs show a Gemini 503 outage outlasting it in 4 of 6 optimizer result files **[Measured]**.
- Under a provider-wide outage, many users retrying in lockstep could make things worse. That is a **[Hypothesis]**; no load test exists.

**Alternatives.** Retry only on specific error classes (429, 5xx, timeouts, parse errors); add jitter; use a circuit breaker; queue the work and retry in a worker; rely on the SDK's built-in retry options (not configured here).

**Related.** Structured output (1), report state machine (6), reliability reasoning in [05](05-engineering-decisions.md).

**Likely interview questions.**
- "Why is jitter important?" Without it, clients that failed together retry together.
- "What would you change about this retry helper?" Narrow the error classes, add jitter and a deadline, and make the wait respect the provider's hints.

---

## 3. The thinking budget

**First principles.** Some models can spend hidden "reasoning" tokens before writing their visible answer. That improves accuracy on tasks with many interacting constraints but costs latency and money. Crucially, those hidden tokens are drawn from the **same output budget** as the visible answer. Give the model a small ceiling and let it think a lot, and the visible JSON gets whatever is left, possibly cut off mid-string.

**Mechanism.** Gemini 2.5 exposes a `thinking_budget`: `0` disables thinking, `-1` lets the model decide dynamically. The project sets it explicitly on every backend call.

| Call | Thinking | Output ceiling | Why |
|---|---|---|---|
| Resume / JD analysis, report, code review, memory notes | off (0) | 4,096 / 16,384 for the report | extraction and summarisation; speed and no truncation |
| Question generation, no coding questions | off | 8,192 | same |
| Question generation with coding questions | dynamic (-1) | 16,384 | measured to stop dropped coding tags |
| Live voice interviewer | not set (model default) | not set | the plugin passes neither unless asked |

**How the project uses it, with evidence.** Two repository-recorded findings drove the settings (details in [05](05-engineering-decisions.md), challenges 2 and 3):
- Report JSON truncated with `Unterminated string` when thinking was left on and the ceiling was 4,096; the fix was thinking off, JSON mime type, and a 16,384 ceiling.
- With thinking off, the model dropped the `coding` tag on some requested coding slots; dynamic thinking fixed it. The commit message says 13/13 compliant after the change versus roughly 1 in 4 slots dropped before; a code comment says "0 misses across 6 trials". Neither can be re-measured without live API access. **[Not measured here]**

**Why it helps.** Reasoning is paid for only where it was observed to help. Elsewhere it is off, which is faster, cheaper and avoids truncation.

**Limitations.** The *why* is empirical: that the per-slot constraints need reasoning is an **[Inference]**. The live interviewer runs on the model's default thinking behaviour and nobody has measured its effect on voice latency. Trace logging records the visible output token count, not thinking tokens, so thinking usage is not directly visible. **[Verified in code]**

**Alternatives.** Always-on thinking; a fixed token budget; moving the "which slots are coding" decision wholly into code and asserting the result (concept 5).

**Related.** Prompt design (4), structured output (1).

**Likely interview questions.**
- "Why did your JSON get truncated?" Hidden reasoning tokens consumed the output budget.
- "How did you decide thinking on or off?" By observation per call site, not by blanket policy; and say honestly which numbers are recorded in the repo and which you cannot reproduce.

---

## 4. Prompt design and conditional prompts

**First principles.** In an LLM application the prompt is business logic: it carries the role, the rules, the output schema and the data. Models are sensitive to everything in the prompt, including parts you meant as irrelevant. Mentioning a concept can *prime* the model to use it.

**Mechanism.** The project assembles prompts from parts. The question-generation prompt is a main template plus a difficulty framework plus a Python-written "required question sequence" plus the two profiles as JSON, plus optional blocks that appear only when relevant: a coding-format block (only if coding questions are requested) and a returning-candidate block (only if memory is opted in). The live interviewer's system prompt is a short intro, the **entire plan as JSON**, and a behavioural rules block. **[Verified]**

**What the model is told versus what code decides.** The difficulty mix, which slots are coding, the fixed warm-up and the fixed closing question are all decided in Python and written into the prompt as an explicit slot list ("slot 3: intermediate [CODING]"). The model only fills in content. Example (as built for 5 slots, one coding): **[Measured]**

```text
slot 1: basic (difficulty=1) - fixed warm-up resume question
slot 3: intermediate (difficulty=2) [CODING]
slot 5: CLOSING - "Do you have any questions for me?"
```

**Why conditional blocks matter: a recorded failure.** Originally the coding-question format was always described. The model then sometimes wrote `category: "coding"` (that word belongs in a different field), which raised a `ValueError` and crashed generation. The fix made the block conditional and added an always-on "never `coding` as category" rule. A test asserts on the assembled prompt text. ([05](05-engineering-decisions.md), challenge 1.) The explanation "irrelevant mentions prime the model" is the commit's diagnosis; no A/B ablation is stored, so treat it as **[Inference]**.

**Why it helps.** Shorter, relevant prompts are cheaper and less error-prone; decisions that must be exact are not left to the model.

**Limitations.**
- No prompt versioning: nothing records which prompt version produced a plan or report.
- The live prompt contains the answer key (`ideal_answer_points`, coding test cases), which is useful for judging follow-ups but is a leakage risk and a token cost resent on every call. Size is about 1,430 characters per question on a synthetic plan, giving roughly 17-29 thousand characters of system prompt **[Measured on synthetic plans, real plans will differ]**.
- Two separately maintained texts (the interviewer rules and the eval judge's rubric) once contradicted each other ([05](05-engineering-decisions.md), challenge 8).
- Cosmetic wart: three concatenated templates contain literal doubled braces because they were written for `.format()` but are not formatted. The model copes.

**Alternatives.** Templating engines, prompt registries with versions, few-shot examples (none used), compiled prompts (DSPy style), a smaller live prompt that fetches question details through a tool.

**Related.** Soft vs hard enforcement (5), memory hygiene (10).

**Likely interview questions.**
- "Show me the most engineered prompt in the project and say what you would test." The question-generation prompt; test the assembled text per configuration and validate the output after generation.
- "What is in the live system prompt and what is the risk?" Whole plan with answer key; leakage and per-turn cost.

---

## 5. Soft versus hard enforcement

**First principles.** There are two ways to make a rule hold. A **soft** rule is an instruction to the model ("never ask more than two follow-ups"); it is a request and the model may ignore it. A **hard** rule is enforced by code that physically prevents the violation ("after two follow-ups, advance automatically"). Between them are **detective** controls that do not prevent anything but measure violations afterwards. A good system layers all three and is honest about which layer each rule is in.

**The project's follow-up cap shows all three layers.** **[Verified]**
1. *Prompt:* the interviewer rules ask for at most two follow-ups.
2. *Code counter plus per-turn note:* each completed candidate turn is counted in Python (the first turn answers the question, later turns answer follow-ups). A temporary system message "Follow-ups used on this question: n/2" is injected before every model reply, with a stop instruction once the cap is reached.
3. *Detective check:* an eval check counts candidate turns per question in transcripts and flags more than three.

The cap is still **soft**: nothing auto-advances; the model can ignore the note. What the code enforces is the *counting*, so the model always sees its exact standing. Four consecutive candidate turns produce notes `0/2`, `1/2`, `2/2 + stop`, `3/2 + stop` **[Measured]**.

**Other examples of "decide in code, let the model fill content".** Level sequence and coding-slot choice (code), opening line (a fixed string spoken without an LLM call), coding editor shown by Python (not left to the model to remember), question order (plan), and test pass counts (concept 9).

**The "guarantee" that is not one.** Difficulty levels are assigned per middle slot by an independent weighted random draw. Over 2,000 simulated 16-question interviews the average shares matched the weights (about 39.9 / 35.1 / 25.0 %), but the number of advanced questions per interview ranged 0 to 11 and 1.8 % of interviews had none **[Measured]**. Code also only *logs* target versus actual counts after generation. So "the code decides levels" is true; "the mix is guaranteed" is not. The README's word "guarantee" (and a code docstring) overstates it.

**Why it helps.** Moving a decision from the model to code gives you determinism where you need it and leaves the model the part it is good at (language).

**Limitations and a trade-off.** The hook that injects the note modifies the model's context each turn, which, per the LiveKit library source, discards the speculative reply the framework had already started, so the latency head start is lost on ordinary questions. Effect **[Not measured]**. See concept 8.

**Alternatives.** Auto-call `move_to_next_question` in code at the cap (hard); a validator that blocks a reply and regenerates; deterministic quota allocation (largest-remainder rounding, then shuffle) for the difficulty mix; post-generation assertions with retry for slot and count compliance.

**Related.** State machine (6), tool calling (7), evals in [06](06-quality-and-reliability.md).

**Likely interview questions.**
- "How do you make an LLM follow a rule?" Layer: say it, count it, measure it, and enforce it in code where it matters; name which layer each rule is on.
- "Is your follow-up limit guaranteed?" No; say exactly what is enforced (counting, not behaviour) and how you would harden it.

---

## 6. The interview as a state machine (and the report as another)

**First principles.** A state machine is a system that is always in exactly one named state and moves between states through defined events. It is the standard way to keep something long-running predictable. The trick for LLM products: let code own the state and transitions, and let the model *request* transitions through tools.

**The interview state.** A single session object holds: the plan, the current question index, asked question ids, per-question counters (candidate turns, follow-ups), the transcript, and start/end times. The transitions are: advance to the next question (resets counters), end the interview (records the end time, starts report generation). Question order never comes from the model. **[Verified]**

**Autosave with coalescing.** Every transcript entry schedules one save to Redis about a second later; entries arriving in that second are included because the save snapshots the current object; a pending save blocks scheduling another. So five entries in a second produce one write containing all five (a unit test asserts this). The benefit: a crash no longer loses the whole transcript, so a report can still be produced. The costs: the last second can be lost, each save writes the **entire** state, and failures are only logged.

An important limit: on (re)start the agent loads **only the plan**, not the saved transcript or counters, and the follow-up counters are not part of what is persisted. That is why "Reconnect" restarts the interview rather than resuming it. See [03-how-it-works](03-how-it-works.md) for the flow.

**The report as a state machine.** Report status moves `pending` to `ready` or `failed`. The pipeline never raises: any failure becomes the explicit state `failed` plus a log line, and the optional memory update sits in its own `try` so it cannot fail the report. The browser polls and shows a Retry button on `failed`; the retry endpoint rebuilds from the autosaved transcript. **[Verified]** Why it matters: the pipeline may run as a background task nobody awaits, so an unhandled exception would vanish and leave the UI polling forever. The state machine converts a silent failure into an observable one.

```mermaid
stateDiagram-v2
    [*] --> pending: interview ends / retry requested
    pending --> ready: report stored
    pending --> failed: generation failed after retries
    failed --> pending: user presses Retry
    ready --> [*]
    note right of pending: no staleness timeout, a crash here leaves it pending
```

**Limitations.** `pending` has no timeout, so a process crash mid-generation strands it (reasoned, not observed). Memory read-modify-write has no lock.

**Alternatives.** A durable job queue with retries and dead-letter handling; an append-only Redis stream for the transcript instead of rewriting a blob; persisting counters and resuming from saved state.

**Related.** Tool calling (7), SSE (11).

**Likely interview questions.**
- "Why model report status as an explicit state?" So failures are visible and retryable instead of silent.
- "Why does reconnect restart the interview?" The agent restores only the plan; this is a known limitation, not a feature.

---

## 7. Tool calling and what kind of "agent" this is

**First principles.** In tool calling you describe functions to the model. Instead of text it may answer "please call `move_to_next_question`". *Your* code runs the function and sends the result back, and the model continues. The model never executes anything. One practical consequence: a tool call costs **two** model calls, one to request and one to read the result.

**Workflow versus agent.** A *workflow* is a code-defined sequence of steps, some of which are LLM calls. An *autonomous agent* picks its own steps in a loop. This project has both ends of the spectrum but nothing fully autonomous: **[Verified]**
- Preparation is a **workflow**: parse, resume analysis, job analysis, question generation, store. Each "sub-agent" is a single LLM call, not a loop.
- The live interviewer is a **tool-calling LLM inside a code-controlled sequence**: three tools (move to next question, end interview, read current question context). The model decides *when* to follow up, advance or end, but the order, counters, opening line and editor display are code. A **constrained agent**.
- There is no RAG, no embeddings, no multi-agent coordination, no agent framework. The whole context (resume, job description, plan) fits in one prompt, so retrieval is unnecessary; that is a design observation, not a recorded rationale.

**The three tools in behaviour terms.**
- *Move to next question:* advances state, saves to Redis, and returns the next question's details and any follow-up hints. If the next question is a coding one, Python also publishes the editor to the browser, and the tool result tells the model to explain briefly and wait for the submission. After the last question it says to end.
- *End interview:* records the end time and fires report generation as a background task that nothing awaits.
- *Get current question context:* read-only.

```mermaid
sequenceDiagram
    participant C as Candidate
    participant P as Voice pipeline
    participant M as Gemini
    participant T as Tools and session state
    C->>P: finishes answering
    P->>M: prompt + history + follow-up note
    M-->>P: request: move to next question
    P->>T: run tool (advance, save to Redis)
    T-->>P: next question details
    P->>M: history + tool result
    M-->>P: spoken text
    P-->>C: speech (streamed)
```

The figure shows why a tool turn is slower: two model round trips. In stored text-mode eval runs some interviewer turns came back empty (7 silent-turn records across 2 of 9 checked runs); whether a tool call preceded them is not recorded, and the mechanism is unknown ([05](05-engineering-decisions.md), observation B).

**Limitations.** Because the whole plan is already in the prompt, tool results repeat question details; the repo records no reason (plausibly to keep the model oriented, an **[Inference]**). The eval harness re-implements the tools as plain functions, kept in sync by hand, and the text-mode version has no coding branch, so the mirror is approximate.

**Alternatives.** Scripted interviewer (no tool calling); fully autonomous agent; function calling for structured output; a smaller prompt with a "fetch question" tool.

**Related.** State machine (6), soft vs hard enforcement (5), voice (8).

**Likely interview questions.**
- "Is this an agent?" A constrained one. Name the guardrails.
- "Why no RAG?" Everything needed fits in one prompt; retrieval would add stages without a benefit. Be ready to describe a RAG pipeline anyway: ingest, chunk, embed, index, retrieve, generate, evaluate retrieval.

---

## 8. Voice turn-taking: endpointing, preemption, interruption

**First principles.** Voice conversation is hard because of *when* to talk. Silence alone is wrong both ways: too short and you cut people off mid-thought, too long and the conversation feels dead. Endpointing is the decision "the speaker has finished". It combines a voice-activity detector (a small model that says "speech / not speech" many times a second) with, optionally, an end-of-turn model that reads the text so far and predicts whether the sentence is complete.

**Mechanism in this project.** The cascade is speech-to-text, then the LLM, then text-to-speech, wired by the LiveKit Agents framework. Our configuration sets only the models (Soniox STT, Gemini Flash, Cartesia TTS, Silero VAD, an optional end-of-turn model); the timing behaviour comes from **library defaults, read from version 1.8.4**, while the requirement allows any 1.x from 1.5, so a rebuilt image may differ. **[Verified]**

- The VAD ends speech after 0.55 s of silence. Endpointing waits at least 0.5 s and at most 3.0 s, measured from the last speech. Because 0.55 s already exceeds 0.5 s, the turn normally commits at about 0.55 s; the wait stretches toward 3 s only when the end-of-turn model predicts "unfinished".
- If the end-of-turn model's files are missing, the code falls back to VAD only (it logs a warning). The agent Dockerfile never downloads those files, so a containerised agent probably runs in the fallback, where a thinking pause longer than about 0.55 s can be treated as a finished answer **[Inference]**; the container log was not checked.
- **Preemptive generation:** the framework may start the LLM request early, on a near-final transcript, so the model's first-token delay overlaps the silence wait. Our follow-up hook adds a message to the context on every non-coding turn, which makes the speculative request non-equivalent and it is discarded and regenerated. On coding questions the hook returns nothing, so the head start can survive. Cost **[Not measured]**.
- **Interruptions:** speaking over the agent for at least 0.5 s cuts its speech; 2 s of silence afterwards counts as a false interruption and it resumes. The opening greeting is uninterruptible, and audio spoken during it is discarded, not queued.

```mermaid
flowchart TD
    A["Candidate speaks"] --> V["VAD: speech start and stop"]
    A --> S["STT: interim, near-final, final text"]
    V --> E["Endpointing (0.5 s to 3 s) + optional end-of-turn model"]
    S --> E
    S -.-> PG["Speculative LLM start"]
    E --> H["Follow-up hook: adds note, invalidates speculation"]
    H --> L["LLM reply or tool call"]
    L --> TTS["Streaming TTS"]
    TTS --> O["Audio to browser"]
```

Read it as the sequence of waiting stages. The reply delay is roughly: endpointing, plus STT finalisation, plus LLM first token (plus a second LLM call on tool turns), plus TTS first audio, plus network. Only the endpointing portion is derivable from configuration; **no end-to-end latency has been measured and no latency metric is collected**. [05](05-engineering-decisions.md) separates reasoned from measured.

**Why a cascade (not speech-to-speech).** Transcripts are first-class: the report, retries, evals and tracing all depend on text, and tools are mature. The original design spec records this "half-cascade" reasoning. The trade-off is the extra latency of three serial stages.

**Limitations.** One transcript entry per final STT segment, so a long answer can produce several entries while the follow-up counter counts whole *turns*; the interviewer's recorded text is what the LLM wrote, not necessarily what was spoken after an interruption; the plugin used for the end-of-turn model is deprecated.

**Alternatives.** Speech-to-speech provider APIs; STT-based turn detection; tuned endpointing delays; keyword-biased STT (supported, unused).

**Related.** Tool calling (7), soft enforcement (5), foundations in [01](01-foundations.md).

**Likely interview questions.**
- "Where does latency come from in a voice agent?" Walk the serial stages; say which you have numbers for and which you do not.
- "How do you avoid cutting a candidate off while they think?" End-of-turn model with a longer maximum wait, plus noting that this project may run without it in Docker.

---

## 9. Grounded code review

**First principles.** Asked "is this code correct?", an LLM can guess, and can also claim tests passed. The fix is the same one a human reviewer uses: run the code. **Grounding** means supplying the model with facts from a deterministic process and letting it do only the part that needs judgement, with the facts having the final word.

**Mechanism.** When the candidate submits code over the LiveKit data channel, the agent calls the review function: **[Verified]**
1. For each test case, append the test's snippet (which calls the candidate's function and prints) to the submitted code, run it in **Piston** (a self-hosted sandbox with a 3-second run limit), and compare trimmed stdout to the expected output; exit code must be zero.
2. Give the model the problem, the code and the *real* results, and ask for quality notes, feedback and a score.
3. **Overwrite** the model's pass counts with the measured ones. Ground truth wins.
4. The agent records the submission and the review in the transcript, then makes the interviewer speak a reaction (ask at most one follow-up, then move on).

The review runs as its own background task so a slow sandbox or LLM call does not block audio.

**Why it helps.** Pass counts are facts, so the model cannot invent them; the expected outputs are never sent to the browser; infinite loops are killed by the sandbox.

**Limitations.**
- If Piston is down, each test becomes a failure and the score looks like a wrong answer; infrastructure failure and wrong code are indistinguishable. Reasoned, not observed.
- Tests run sequentially, one HTTP call each; the candidate hears nothing while they run.
- The review call is not retried; a failure triggers a spoken apology, and the apology asks the candidate to resubmit although the editor has already been hidden, so there is no way to resubmit. **[Verified by reading both sides]**
- The "one follow-up" rule after a review is prompt-only, and submissions are trusted by question id only.
- A Piston container needs elevated privileges to build its own sandbox: a security trade-off.
- The data message has no acknowledgement; a lost submission is silent.

**Alternatives.** LLM-only review; a hosted judge service; local `exec` (unsafe); retry plus an explicit "infrastructure error" outcome.

**Related.** Soft vs hard (5), data-channel flow in [03](03-how-it-works.md).

**Likely interview questions.**
- "How do you stop the model lying about tests?" Run them, overwrite the numbers.
- "What happens when the sandbox is down?" Today it looks like failing code; I would separate infrastructure errors from wrong answers.

---

## 10. Candidate memory and prompt-injection hygiene

**First principles.** *Prompt injection* is text in your data that tries to act as an instruction to the model. The risk compounds when model output is stored and later fed back as input: a **two-hop** path. Defences are about keeping data out of instruction positions and limiting blast radius; none is absolute.

**Memory mechanism (no embeddings, no retrieval).** If a candidate opts in with an email, the system keeps, per hashed email, per-category score history, up to three "weak categories" (those whose average is below 6.0, weakest first), the last 50 asked questions, and a 2-3 sentence LLM-written note. The next preparation injects a summary into the question-generation prompt only: weak areas, a practice-mode quota ("at least two thirds of the middle questions from weak categories"), up to 15 recent questions to avoid, and the notes. **Memory is deliberately never put into the live interviewer's prompt**, a smaller injection surface. **[Verified]**

**Why averages.** One bad session should not permanently label a category weak, nor one good session cure it. Exactly 6.0 is not weak, 5.99 is **[Measured]**. Only a short summary is injected because prompts have a size and attention budget.

**The injection hardening.** The notes come from a summary derived from a transcript the candidate influences, then reappear in the next prompt. The stored notes are therefore wrapped in `<candidate_notes>` tags with the explicit wording "untrusted historical data, not instructions". A test shows that a note saying "Ignore all previous instructions..." is retained but framed as data. The commit itself says this does not claim to block a determined attacker. **[Verified]**

**Limitations.**
- No sanitisation; no classifier; resume and job-description text also goes unsanitised into analysis prompts.
- Notes can persist for up to a year; the memory key is a hash of the email, which hides the key but not guessable emails, and anyone who knows an email can read or alter its memory (the README calls this an accepted trade-off).
- Read-modify-write without a lock can lose an update in rare concurrent cases.
- Report scores for strong candidates cluster near the top (stored runs: mean 9.19 over 12 strong runs **[Measured]**), so weak categories may never trigger.
- Report categories (including `communication`) are not the same set as question categories, so communication becomes qualitative guidance rather than a topic.

**Alternatives.** Accounts and auth; sanitise or classify notes; store only numbers (no free-text notes); embeddings plus retrieval for richer memory.

**Related.** Prompt design (4), report state machine (6).

**Likely interview questions.**
- "Where is prompt injection possible here?" Resume/JD text, spoken answers, and stored notes; explain the two-hop case and why the live prompt excludes memory.
- "Is delimiting enough?" No; it reduces risk. Describe defence in depth.

---

## 11. Server-Sent Events (SSE) streaming for preparation

**First principles.** An HTTP response normally ends when its body is complete. A *streaming* response keeps the connection open and sends pieces as they are ready. SSE is the simplest standard framing: each message is a line starting with `data: ` and ended by a blank line. Preparation makes three sequential LLM calls, so showing progress keeps the user from staring at a frozen page.

**Mechanism.** The preparation endpoint is an async generator that yields progress events at milestones (5, 15, 20, ... 100 percent), then a final complete event with the session id. The "protocol" is: serialize a dict as JSON, prefix `data: `, append `\n\n`. Because the browser's built-in `EventSource` only supports GET, the frontend uses `fetch` with POST and parses the stream by hand with a **buffer**: bytes can arrive split anywhere, so it appends each chunk, splits on newlines, keeps the last (possibly incomplete) piece, and only parses complete lines. **[Verified]**

**Error discipline.** After the first byte is sent the HTTP status can no longer change, so failures must travel *inside* the stream as an error event. A dedicated exception type marks messages safe to show users (bad file type, oversized input). Any other exception becomes a generic message while the traceback goes only to the logs, so internals do not leak. **[Verified]**

**Why it helps.** Progress feedback with no job queue and no extra endpoint.

**Limitations.** One-way; no event ids and no resume, so a dropped connection loses the result (the plan may already be in Redis, but the browser does not know). The hook has no "stream ended without a complete event" timeout, so a silent drop leaves the progress bar frozen. Resume parsing runs synchronously on the event loop, which could stall other requests (**[Not measured]**). SSE is a UX choice, not a speed-up.

**Alternatives.** WebSocket; polling with a job id (more robust to disconnects); a background job plus status endpoint.

**Related.** Report polling (6): the report is produced in a different process after the browser has left the room, so Redis plus HTTP polling is the cheap bridge. WebSockets there would need a server-side subscription mechanism that does not exist (**[Inference]**).

**Likely interview questions.**
- "Why not `EventSource`?" GET only; this endpoint takes a file upload via POST.
- "Why does the client need a buffer?" TCP does not respect message boundaries.

---

## 12. Explicit agent dispatch and signed tokens (JWT)

**First principles.** A JSON Web Token is `header.claims.signature`, where the signature is an HMAC over the first two parts using a secret shared by the issuer and the verifier. Whoever holds only the token cannot alter its claims without invalidating the signature, and the verifier needs no network call to check it.

**Mechanism.** The backend mints a LiveKit access token locally (no call to LiveKit). Its claims include the participant identity and name, a two-hour expiry, a *grant* limited to one room (publish, subscribe, publish data), and a **room configuration that requests an agent by name** with metadata containing the session id. When the browser connects, LiveKit creates the room and asks a registered worker named `mockmind-interviewer` to take a job with that metadata; the agent reads the session id and loads the plan from Redis. Nothing else connects the browser to the agent. **[Verified]**

**Why it helps.** One browser request starts everything; dispatch is *explicit* (only rooms asking for that agent get it); the metadata is signed, so the browser cannot request a different session or agent; least-privilege grants (one room, two hours).

**Limitations.**
- There is no caller authentication: anyone with a session id can mint a token (capability URL model).
- Misconfiguration fails quietly: a missing worker or wrong agent name produces no agent, and the UI only shows a timeout overlay after 25 s. Missing server env vars cause a `KeyError` (HTTP 500).
- Token lifetime is two hours but the plan lives four, so a candidate may get a token after two hours but not after four **[Verified]**.
- Same identity for repeated joins; whether LiveKit then drops the older connection was not tested (**[Hypothesis]**).
- The design spec showed a server-side dispatch call instead; the code used the token form from its first commit; no reason is recorded.

**Process model (what happens next).** Each job runs in its own OS process by default; the worker keeps a few warm processes and stops accepting jobs above a load threshold (library defaults, 1.8.4). Capacity per machine is the number of job processes that fit; not measured end to end.

**Alternatives.** Server-side dispatch call after the room exists; automatic dispatch to every room; an authenticated session service.

**Related.** Shared state in [02-architecture](02-architecture.md); join sequence in [03](03-how-it-works.md).

**Likely interview questions.**
- "How does the agent get into the right room?" Explain the token's room configuration and the metadata.
- "Why can't a user forge it?" HMAC signature; the claims are covered.

---

## Recap

- Treat LLM output as untrusted: parse, validate, retry (1, 2). Spend reasoning tokens only where measured to help, and mind that they share the output budget (3).
- Prompts are logic: keep them relevant and conditional, and move exact decisions into code (4, 5). Be precise about soft versus hard.
- The model proposes, code disposes: state machines, explicit failure states, tool calling inside a fixed sequence (6, 7).
- Voice is a serial cascade whose weakest knowledge here is latency; turn-taking depends on library defaults and on an optional model that may be absent in Docker (8).
- Ground judgements in facts (9); treat stored AI text as untrusted (10); report errors inside streams (11); sign what the browser carries (12).
