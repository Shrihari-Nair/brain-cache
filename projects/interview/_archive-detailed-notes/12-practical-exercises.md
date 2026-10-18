# 12 — Practical Exercises

**Rules.** Exercises never require changing the real application. Write throwaway scripts in a scratch directory outside the repo and import repo modules from there (`sys.path.insert(0, "<repo>/mockmind")`, run with `mockmind/backend/.venv/bin/python`). Replace/patch functions only inside your own process. Try first, open **Hint**, then **Solution**.

**How the solutions were produced.** Every exercise marked **[Ran offline]** was executed by me with the repo venv in a scratch directory (no network, no paid API, no repo edits), and the output in its solution is the real observed output (trimmed only where stated). Exercises marked **[Needs live APIs]** or **[Design only]** were **not** run and their solutions make no result claims. Dummy values (`dummykey`, `dummy-not-a-key`) stand in for secrets; never put real keys in scripts.

**Baseline [Measured]:** `cd mockmind && backend/.venv/bin/python -m pytest backend/tests -q` → `53 passed, 1 skipped, 17 warnings in 1.02s`.

**Plain-words prerequisites.** *Async function*: a function that can pause (`await`) while waiting for I/O so others run. *Mock/stub*: a fake replacement for a slow or paid dependency. *TTL*: seconds after which Redis deletes a key. *Histogram*: counts of how often each value occurred. *Cohen's kappa*: agreement between two labelers corrected for chance.

Numbering note: other chapters cite exercises by number (1.1–1.7, 2.1–2.7, 3.1–3.4); "exercise A3" in 02c means **3.6** here.

---

## Level 1: Beginner (read, run, trace)

### 1.1 Run the tests; predict a retry count [Ran offline]
**Objective:** run the suite and learn what `retry_async` does on permanent failure. **Prerequisites:** Python venv exists; know what an exception is.
**Do:** predict how often an always-failing function is called by default, and which sleeps happen. Check with a script that records sleeps instead of waiting.
<details><summary>Hint</summary>Default `attempts` in `services/llm_retry.py`; patch `llm_retry.asyncio.sleep` to a recorder in your own script.</details>
<details><summary>Solution</summary>

Observed: `raised: boom` / `calls: 3 sleeps: [1.0, 2.0]`, with log lines "failed (attempt 1/3) … retrying in 1.0s", "(attempt 2/3) … 2.0s", "failed after 3 attempts". The last exception is re-raised. The existing test uses `base_delay=0`, so the real 1 s/2 s timing is not asserted anywhere.
</details>

### 1.2 Redis key map [Ran offline: grep]
**Objective:** list every Redis key, writer, reader, TTL. **Prerequisites:** `grep`.
<details><summary>Hint</summary><code>grep -rn "setex\|redis_client.get\|r.get(\|\.delete(" backend agent --include=*.py</code> (exclude tests and `.venv`).</details>
<details><summary>Solution</summary>

Observed hits: `prepare.py` L186/L194 `setex` (plan, session state, 14 400 s); `session.py` L37 plan `get`, L97 report `get`, L118/L121 report and `session_state` `get`; `report_pipeline.py` L26 `setex` report; `report_store.py` L17/L21 status `setex`/`get` (86 400 s); `memory_store.py` L16/L23/L27 get/setex/delete (1 year); `agent/session_state.py` L39 plan `get`, L128 `setex` session state (14 400 s). Table: plan (backend writes, agent+token read, 4 h); session_state (backend writes empty, agent overwrites, retry route reads, 4 h); report and status (agent via shared pipeline or retry route write, report endpoint reads, 24 h); memory (report pipeline writes, prepare reads, forget-me deletes, 1 year).
</details>

### 1.3 Trace `POST /api/token` on paper [Design only; see 2.5 for the run]
**Objective:** mark every raise point. **Prerequisites:** reading FastAPI handlers.
<details><summary>Hint</summary><code>routers/session.py</code> L28–83.</details>
<details><summary>Solution</summary>404 if the plan key is absent (L37–42); `KeyError` (HTTP 500) per missing env var `LIVEKIT_API_KEY/SECRET/URL` (L44–46); otherwise a 2-hour HS256 JWT with room-join grant and an agent dispatch carrying `{"session_id": …}`. Exercise 2.5 confirms by running.</details>

### 1.4 The soft guarantee [Ran offline]
**Objective:** see exactly what the model is told, and why it is not forced to stop. **Prerequisites:** none.
**Do:** call `follow_up_policy_note` four times on one non-coding question, then on a coding question.
<details><summary>Hint</summary>Build state with `AgentSessionState("s", make_plan(n_questions=3))` from `backend/tests/conftest.py`.</details>
<details><summary>Solution</summary>

Observed: turn 1 `Follow-ups used on this question: 0/2.`; turn 2 `1/2.`; turn 3 `2/2. The limit is reached: do NOT ask another follow-up. Briefly acknowledge the candidate's answer, then call move_to_next_question.`; turn 4 `3/2. The limit …` (the counter keeps rising); after `advance_question()` counters are 0/0; coding question returns `None`. The function returns text only; no code path blocks a model reply.
</details>

### 1.5 Input limits and presets [Ran offline]
**Objective:** learn the upload rules and presets. **Prerequisites:** none.
<details><summary>Hint</summary><code>validate_resume_upload</code>, <code>resolve_plan_options</code> in <code>routers/prepare.py</code>.</details>
<details><summary>Solution</summary>

Observed: `cv.pdf` 1000 B ok; `cv.PDF` at exactly 5 242 880 B ok; 5 242 881 B → "Resume is too large (max 5 MB)."; 0 B → "The uploaded resume is empty."; `cv.exe` and `cv` → "Unsupported file type…"; `notes.doc` ok. Limits 5 242 880 / 30 000 / 15 000. Presets: standard/balanced/standard → 16 questions, 0 coding, weights `None`; harder+quick → 8 questions, weights 0.15/0.35/0.5; coding+easier → 6 questions, 4 coding, weights 0.6/0.3/0.1; unknown values → defaults.
</details>

### 1.6 What does `min_pass_rate=0.8` mean with 3 runs? [Ran offline]
**Objective:** reason about small samples. **Prerequisites:** fractions.
<details><summary>Hint</summary><code>evals/summary.py</code> <code>summarize</code>, <code>below_threshold</code>.</details>
<details><summary>Solution</summary>

Observed: 0/3 → 0.0 below; 1/3 → 0.333 below; 2/3 → 0.667 below; 3/3 → 1.0 not below. Only a perfect 3/3 passes, so one flaky run fails the job. A rate from 3 runs is a smoke signal, not an estimate.
</details>

### 1.7 Tracing is a no-op when unconfigured [Ran offline]
**Objective:** confirm graceful degradation. **Prerequisites:** context managers.
<details><summary>Hint</summary>Unset `LANGFUSE_PUBLIC_KEY`/`LANGFUSE_SECRET_KEY` in your script, then use `traced_generation`.</details>
<details><summary>Solution</summary>

Observed: `is_enabled: False`, `handle: _NoopGenerationHandle`, and an exception raised inside the `with` block still propagates (`KeyError('inside')`).
</details>

---

## Level 2: Intermediate (script against repo modules)

### 2.1 What does `validate_report` really accept? [Ran offline]
**Objective:** find untested or missing rules. **Prerequisites:** 1.x; reading `report_generator.py` L41–58 and `test_report_validation.py`.
**Do:** feed ten crafted dicts and predict accept/reject first.
<details><summary>Hint</summary>Compare each `raise` with the six existing tests; then try values the function never inspects.</details>
<details><summary>Solution</summary>

Observed: valid → accepted (overall `7.5`, `question_feedback=[]` default); missing `overall_score` → `ValueError: report missing keys`; bad recommendation → `ValueError`; `category_scores` a list → `ValueError: category_scores must be an object` (rule exists; the 09 chapter notes no test covers it); `"high"` → `ValueError: could not convert string to float` ; `None` → **`TypeError`** (not a clean ValueError; still retried because `retry_async` catches all); **`overall_score: 99` accepted** (no range check); `category_scores: {"technical": "great"}` accepted; `question_feedback: "oops"` accepted as a string; a list → `ValueError: report is not a JSON object`. Gap: scores/ranges and per-question entries are unvalidated, so the report page could still receive odd data.
</details>

### 2.2 Difficulty draw is random, not a quota [Ran offline]
**Objective:** measure the distribution behind the README's "guarantee". **Prerequisites:** histogram idea.
<details><summary>Hint</summary>Seed `random`, call `_assign_level_sequence(16, weights)` 10 000 times; `_normalize_level_weights(None)` gives the defaults.</details>
<details><summary>Solution</summary>

Observed weights `{'basic': 0.4, 'intermediate': 0.35, 'advanced': 0.25}`. Count of "advanced" among 16 questions (seed 0, 10 000 draws): 0:162, 1:800, 2:1838, 3:2388, 4:2217, 5:1457, 6:759, 7:282, 8:78, 9:17, 10:2; mean 3.51 vs expected 3.5; 1.6 % of interviews had none. So the mix is a random draw per interview; the model's compliance is a separate, unchecked matter.
</details>

### 2.3 Retry storm and jitter [Ran offline]
**Objective:** show lockstep retries. **Prerequisites:** `asyncio.gather`.
<details><summary>Hint</summary>Patch `llm_retry.asyncio.sleep` with a function that records a per-task virtual time.</details>
<details><summary>Solution</summary>

Observed with 100 concurrent callers each failing twice: wake times `[(1.0, 100), (3.0, 100)]`, i.e. all 100 retry at the same instants. A scratch full-jitter formula (`uniform(0, base·2^k)`) spread the 200 retries over 28 distinct 0.1 s buckets. This is the mechanism behind 07 D2; it is a simulation, not an observed outage.
</details>

### 2.4 Add a deterministic check (scratch) [Ran offline]
**Objective:** write a pure check over transcripts. **Prerequisites:** transcript entry shape `{speaker,text,timestamp,question_index}`.
<details><summary>Hint</summary>Model it on `evals/checks.py`; do not edit that file.</details>
<details><summary>Solution</summary>

My scratch check (interviewer turns longer than N sentences) returned `(True, [])` for a good transcript and `(False, [(0, 6)])` for a six-sentence turn; the repo's own `deterministic_checks` on the good transcript returned all three keys `True`. Design note: keep checks pure and fixture-tested.
</details>

### 2.5 Mint a token offline and read it [Ran offline]
**Objective:** verify the token claims. **Prerequisites:** JWT = signed JSON; PyJWT in the venv.
**Do:** call `session.get_token` with a fake Redis and **dummy** LiveKit env values (set inside the script).
<details><summary>Hint</summary>Replace `session.redis_client` with `FakeRedis` from `backend/tests/conftest.py`; decode with `jwt.decode(..., algorithms=["HS256"], options={"verify_aud": False})`.</details>
<details><summary>Solution</summary>

Observed: missing env → `KeyError('LIVEKIT_API_KEY')`; with dummies: room `interview_abcdef12-3456`, identity `candidate_abcdef12`; `exp - nbf = 7200` s; `video` grants `roomJoin/canPublish/canSubscribe/canPublishData` for that single room; `roomConfig.agents = [{agentName: mockmind-interviewer, metadata: {"session_id": …}}]`; unknown session → `404 Session not found. Please run /api/prepare first.`
</details>

### 2.6 Run the preparation stream offline [Ran offline]
**Objective:** see the SSE event sequence. **Prerequisites:** async generators.
<details><summary>Hint</summary>Patch `prepare.analyze_resume`, `analyze_jd`, `generate_questions`, `redis_client` in your process; call `prepare_interview(...)` with a Starlette `UploadFile` and iterate `resp.body_iterator`.</details>
<details><summary>Solution</summary>

Observed stages/progress: parsing 5, parsing 15, resume_analysis 20, resume_analysis 45, jd_analysis 50, jd_analysis 65, question_gen 70, question_gen 90, complete 100; Redis keys afterwards: `interview_plan:<uuid>` and `session_state:<uuid>` only.
</details>

### 2.7 Report pipeline outcomes [Ran offline]
**Objective:** see ready/failed and the stranded `pending`. **Prerequisites:** 2.6 style patching.
<details><summary>Hint</summary>Patch `redis_client` in `report_pipeline` and `report_store`; patch `generate_report`.</details>
<details><summary>Solution</summary>

Observed: success → returns `True`, status `ready`, report key stored; failure (`RuntimeError`) → returns `False`, status `failed`, no report key (traceback logged); a manually set `pending` stays `pending` (TTL 86 400 s): nothing in the code moves it on, which is the 07 D3 hypothesis shown as state, not as a real crash.
</details>

---

## Level 3: Advanced (analysis, measurement, design)

### 3.1 Scripted interview through the real driver [Ran offline]
**Objective:** see how transcript entries are tagged with `question_index`. **Prerequisites:** 2.6; reading `evals/interview_driver.py`.
<details><summary>Hint</summary>Set a dummy `GOOGLE_API_KEY` in the script, patch `interview_driver.genai.Client` with a fake whose `aio.chats.create(...)` returns a chat that calls the tools in `config.tools`, and patch `CandidateSimulator`.</details>
<details><summary>Solution</summary>

Observed (3-question plan): `0 interviewer Hi! Question 1?` · `0 candidate my answer` · `1 interviewer Next: question 2` · `1 candidate` · `2 interviewer Next: question 3` · `2 candidate` · `3 interviewer Next: question 4` · `3 candidate` · `3 interviewer Goodbye.` The tool advances the index *before* the interviewer's next line is recorded, so that line carries the new index, and the closing line carries `N`.
</details>

### 3.2 Test the memory update with a stub [Ran offline]
**Objective:** prove the merge logic without Gemini. **Prerequisites:** 2.7.
<details><summary>Hint</summary>Replace `memory_updater._regenerate_notes` with an async stub; call `update_memory_from_report(existing=None, email=…, session_id=…, plan=…, report=…)`.</details>
<details><summary>Solution</summary>

Observed after session 1 (scores technical 5, behavioral 8, communication 4): `weak: ['communication', 'technical']`, history lists, notes `stub notes`, 3 asked questions. After session 2 with 9/8/9 on the same email (different case): `weak: []`, histories `[5.0, 9.0]`, `[8.0, 8.0]`, `[4.0, 9.0]` (averages 7, 8, 6.5 are all ≥ 6.0).
</details>

### 3.3 Calibrate the judge [Design only; kappa formula ran offline on made-up labels]
**Objective:** design an agreement study for `stayed_in_character`. **Prerequisites:** 1.6.
<details><summary>Solution</summary>

Collect ≥30 transcripts; two humans label each; compute human-human kappa and human-judge kappa; revise the rubric where they disagree; report intervals. Formula check on **invented** labels (not data about this system): 10 items, observed agreement 0.9, chance agreement 0.74, kappa 0.615; an always-"pass" judge against the same humans gets kappa 0.0 despite 80 % raw agreement, which is why raw accuracy misleads. Running the real judge needs live Gemini; **not run**.
</details>

### 3.4 Capacity test plan (two pieces measured offline) [Partly ran offline; ramp needs live APIs]
**Objective:** find max concurrent interviews per agent node. **Prerequisites:** 08 §8.5.1.
<details><summary>Hint</summary>Start with what needs no provider: time Silero VAD on synthetic audio and read process RSS.</details>
<details><summary>Solution</summary>

Observed on this 12-core machine (agent venv, LiveKit Agents 1.8.4): Silero VAD on 30 s of synthetic noise, 10 ms frames: CPU 1.225 s (first run, includes warm-up), then 1.063, 1.066, 1.034 s → **0.035–0.041 CPU-s per audio-second**; 937 events. RSS: 9 MB bare Python; 181 MB after importing LiveKit + plugins; 205 MB after `VAD.load()`; 209 MB after importing `agent.interviewer`. These are floors, not per-job cost (no STT/TTS sockets, no turn-detector model; the model could not be constructed outside a job context). Full plan (not run, needs paid keys): ramp N rooms with a scripted audio source; record per-turn latency percentiles, CPU and RSS per job process; stop when p95 crosses a threshold or LiveKit stops accepting jobs (`load_threshold` 0.7).
</details>

### 3.5 Threat model [Design only; one fact ran offline]
**Objective:** top-5 risks with mitigations. **Prerequisites:** 03 trust boundaries.
<details><summary>Solution</summary>

Expected: no auth/ownership; prompt injection via resume/JD/speech/memory notes; unauthenticated, snapshot-only Redis; PII in Langfuse traces; uncapped LLM spend. Offline fact: two different callers using the same `session_id` each got a token with identity `candidate_abcdef12`; nothing checked who they were (exercise 2.5 script, last line). Compare with 07 C3, C15, D9.
</details>

### 3.6 Instrument voice latency (02c's "exercise A3") [Needs live APIs; Design only]
**Objective:** produce p50/p95 for end-of-speech → first audio. **Prerequisites:** 08 §8.1.2.
<details><summary>Solution</summary>

Design: subscribe to the session's `metrics_collected` events (the library emits them; `on_user_turn_completed_delay` is recorded) and log per turn: end-of-utterance delay, transcription delay, LLM TTFT, TTS first-audio, split by "tool turn" vs not and by coding vs conversational. Compare two arms: current config vs `thinking_budget=0` on the live LLM. **No result is claimed**; it needs real keys and real calls.
</details>

---

## Self-assessment checklist
- [ ] I can list the five Redis key families and who writes each (1.2).
- [ ] I can say which parts are workflow vs agent.
- [ ] I can state three eval findings and label each by evidence level.
- [ ] I can name five things not measured, and the two things I did measure for capacity (3.4).
