# 08 — Performance, Reliability, and Scaling

> **Honesty rule for this chapter.** No load tests, latency benchmarks, or throughput numbers exist in the repository, and I ran no paid or networked call. Every claim carries a label:
> **[Verified]** read in this repo's code/config · **[Measured]** computed or timed locally on files/code with no network · **[Library source]** read in the installed library (LiveKit Agents 1.8.4, google-genai in `backend/.venv`) · **[Standard knowledge]** general engineering fact · **[Inference]** reasoned from the above · **[Hypothesis]** plausible, untested · **[Not measured]**.
> Anything labelled Inference/Hypothesis/Not measured must not be quoted as fact in an interview.

### Plain-words prerequisites
| Term | Meaning |
|---|---|
| **Latency** | how long one thing takes (here: from the candidate finishing a sentence to hearing the reply start). |
| **p50 / p95** | the median, and the "1 in 20 is slower than this" value. Averages hide bad cases; p95 shows them. |
| **Time to first token (TTFT)** | how long an LLM takes to start answering, before the rest streams out. |
| **Timeout** | a limit after which we stop waiting. Without one, a stuck call can wait for a very long time. |
| **Retry with backoff** | try again after waiting 1 s, then 2 s, … so a brief outage can pass. **Jitter** = random extra wait so many clients do not retry at the same instant. |
| **Quota / rate limit** | the provider's cap on requests or tokens per minute for your account. |
| **Snapshot (RDB)** | Redis periodically writes all its data to disk; anything written after the last snapshot is lost if Redis dies. |
| **Token** | a chunk of text (roughly 4 English characters) that LLMs read and bill for. This chapter uses `chars / 4` as a crude estimate and says so. |

## 8.0 Corrections applied in this revision
- **Redis persistence was overstated.** The compose `redis` service (`redis:7-alpine`, no volume, no command) uses the image defaults: RDB snapshots on (`save 3600 1 300 100 60 10000`), AOF off, anonymous `/data` volume (**[Measured]** earlier on the running container, see [02a](02a-backend-and-data-technologies.md) §2a.4 and [07](07-engineering-challenges.md) C4). So "a restart loses everything" is wrong. Correct statement: **writes since the last snapshot can be lost on a crash/kill** (up to about an hour on a lightly written instance **[Inference]** from the rules: 1 change in 3600 s, 100 in 300 s, 10 000 in 60 s), and **the anonymous volume may not survive `docker compose down -v` or container re-creation**, which drops plans, reports and memory. TTLs and key semantics still apply on top.
- **Port note [Verified]:** compose publishes the frontend as `3002:3000`; README and the `CORS_ORIGINS` default in `backend/main.py` assume `http://localhost:3000` (07 C28).
- **No `.dockerignore` [Verified]:** both Dockerfiles do `COPY . .` from `mockmind/`, so a local build copies in `.env` and `frontend/node_modules`. Local-build hygiene issue, not a repo leak (07 C27).
- **Wording fix:** `code_executor.execute_code` takes `run_timeout_ms` (default 3000, `code_executor.py` L50) and sends it as `run_timeout` (L65); the HTTP client timeout is `total=30` s (L57). The old text said "`run_timeout` default".

---

## 8.1 Where time goes in each flow

### 8.1.1 Preparation (`POST /api/prepare`, SSE)
Steps (see [05](05-end-to-end-workflows.md)): validate → parse resume → resume analysis (LLM) → JD analysis (LLM) → memory lookup (optional) → question generation (LLM) → two Redis `SETEX` (plan, empty session state; 14 400 s each, `prepare.py` L186–198) → `complete` event.
- Three **sequential** LLM calls **[Verified]**; total ≈ sum of the three. Resume and JD analyses do not depend on each other, so they could run with `asyncio.gather` **[Inference]**; not implemented; gain **[Not measured]**. The original spec also said "sequentially" (`MOCK_INTERVIEW_PLATFORM.md` L635).
- Coding preset enables dynamic thinking for question generation (`thinking_budget=-1`, `question_generator.py` L332) and a 16 384-token output ceiling (else 8 192, L322) → slower/costlier than the standard path **[Verified config; delta Not measured]**.
- Resume parsing (`extract_resume_text`, `prepare.py` L129) is synchronous CPU work on the event loop; effect on concurrent requests **[Not measured]**.
- Worst-case wall time per LLM step = `3 × T + 3 s` (three attempts plus 1 s and 2 s waits) where `T` is one call's duration, and **no per-call timeout is set anywhere** (§8.8). **[Inference from `llm_retry.py` + grep]**
- SSE is a UX choice (progress messages), not a speed-up.

### 8.1.2 Anatomy of one voice turn (config values actually in effect)
`agent.py` configures only: Soniox `stt-rt-v4` with `language_hints=["en"]`; `google.LLM(model="gemini-2.5-flash", temperature=0.7)`; `cartesia.TTS(model="sonic-3", voice=<fixed id>)`; `silero.VAD.load()` with no arguments; `TurnHandlingOptions(turn_detection=_load_turn_detector())` (L200–219) **[Verified]**. Everything else below is a **library default [Library source]** for 1.8.4; the requirement is `~=1.5`, so a Docker build may resolve a different 1.x (02c).

| # | Stage | Value in effect | Source |
|---|---|---|---|
| 1 | VAD end-of-speech | `min_silence_duration` **0.55 s**, `min_speech_duration` 0.05 s, `prefix_padding_duration` 0.5 s, `activation_threshold` 0.5 | `plugins/silero/vad.py` L63–67 |
| 2 | Endpointing | not set → `min_delay` **0.5 s**, `max_delay` **3.0 s** (the tighter 0.3/2.5 set applies only to a *streaming* detector) | `agents/voice/turn.py` L137–145 |
| 3 | End-of-turn model | `MultilingualModel()` if its files load, else `None` (VAD-only). Docker image likely has no files (**[Inference]**, 02c §2c.6; container log not checked) | `agent.py` L119–127 |
| 4 | STT | Soniox `stt-rt-v4` (library default would be `stt-rt-v5`); `max_endpoint_delay_ms` default 2000 (allowed 500–3000); emits interim, preflight, final transcripts | `plugins/soniox/stt.py` L112, L124, L145 |
| 5 | Preemptive generation | enabled; `preemptive_tts` False (LLM only); skipped if user speech > **10 s**; up to **3** attempts | `turn.py` L225–227 |
| 6 | Our hook | `on_user_turn_completed` adds a system note on every non-coding turn → speculative reply discarded, fresh LLM request (02c §2c.7). Hook cost itself: a counter, microseconds | `interviewer.py` L70–79 |
| 7 | LLM | Gemini 2.5 Flash, temp 0.7. **No `thinking_config` and no output cap passed** (the plugin parameter defaults to `NOT_GIVEN`), so the model's own default thinking behaviour applies **[Library source; effect on TTFT Not measured]**. Contrast: every backend call site sets thinking explicitly. | `plugins/google/llm.py` L148 |
| 8 | Tool turn | `move_to_next_question` → Redis save (new connection each call) → **second** LLM request to speak; `max_tool_steps` = 3 | `interviewer.py` L81–91; 02c |
| 9 | TTS | Cartesia `sonic-3`, `pcm_s16le`, 24 kHz, word timestamps on, streaming | `plugins/cartesia/tts.py` L91–99 |
| 10 | Interruptions | enabled; ≥0.5 s overlap cancels speech; 2 s silence afterwards = false interruption → resume. Opening greeting uses `allow_interruptions=False` | 02c §2c.8; `interviewer.py` L65 |
| 11 | Per-service retry | STT, LLM, TTS each: `max_retry` 3 (first retry after 0.1 s, then 2.0 s apart), `timeout` 10 s per attempt; session closes after more than 3 consecutive unrecoverable errors per kind | `agents/types.py` L124–157; `voice/agent_session.py` L172–176, L1990–2020 |

**Serial formula [Inference]:** `reply delay ≈ T_endpoint + T_STT_final + T_LLM_first_token (+ T_tool + T_LLM_second_call on tool turns) + T_TTS_first_audio + T_network`.
- `T_endpoint` is knowable: about **0.55 s** when the sentence clearly ended (VAD's own 0.55 s already exceeds `min_delay` 0.5), up to **~3.0 s** after the last word if the end-of-turn model is loaded and predicts "unfinished"; with no model loaded it stays ~0.55 s, so a thinking pause longer than that can be cut off (02c §2c.6).
- **Preemptive head start:** on coding questions the hook returns `None`, so a speculative LLM request started at the preflight transcript can be kept and its TTFT overlaps the silence wait. On ordinary questions our note invalidates it, so LLM TTFT is *added* after the endpoint **[Library source reasoning; magnitude Not measured]**.
- **Not known for this system:** STT finalization, Gemini TTFT with default thinking, tool-turn double round trip, TTS first audio, network. No metric, SLO or log of any of these exists; Langfuse spans may carry some if configured (`_setup_otel_tracing` in `agent.py`), unread here **[Verified]**.

### 8.1.3 Report generation
One LLM call: `max_output_tokens=16384`, thinking off (`report_generator.py` L94–96) **[Verified]**, wrapped in `retry_async`. Output length scales with the transcript because every answer is copied into `question_feedback[].candidate_answer` (07 C6). Sizes in §8.7; wall-clock **[Not measured]** (stored results carry no timing).

---

## 8.2 Concurrency model and its limits
(Per-turn latency anatomy is §8.1.2.)
- **Backend:** one asyncio loop per Uvicorn process; compose starts one process with `--reload` and no `--workers` **[Verified]**. All network I/O is awaited; blocking spots: resume parsing, and any CPU-heavy work.
- **Gemini client:** a **new `genai.Client` is built inside every call** (all call sites) **[Verified]**; no semaphore, no limiter, no `Retry-After` handling, no `wait_for`/timeout in app code (grep over `backend/` and `agent/`) **[Verified by grep]**. SDK-level: `HttpOptions.timeout` defaults to `None` and the SDK's own retry runs only if `retry_options` is passed, which we never do **[Library source: `google/genai/types.py`, `_api_client.py` L248–258, L543–560]**. So the only retry is ours (3 attempts) and a hung request has no client-side deadline **[Inference]**.
- **Redis:** backend shares one pooled module-level client (`redis_client.py`); the **agent opens a new connection per `load()`/`save()`** and closes it (`session_state.py` L32–45, L112–134) **[Verified]**.
- **Agent:** one LiveKit **job process per room**; `start` mode keeps `min(CPUs, 4)` warm processes and stops accepting jobs above `load_threshold` 0.7 **[Library source]** (02c §2c.2). Capacity per machine = how many job processes fit; not measured end to end (see §8.5.1 for the two pieces I could measure).
- **Piston:** separate privileged container; per-test `run_timeout` 3000 ms, HTTP `total=30` s; test cases are executed **one HTTP call each, sequentially** (`code_judge.py` L75–104) **[Verified]**.
- **Background tasks:** `end_interview` fires `asyncio.create_task(generate_and_store_report(state))` without keeping a reference (`interviewer.py` L149); the backend retry route does keep references in `_retry_tasks` (`session.py` L109, L137–138). Python docs warn that an unreferenced task can be garbage-collected **[Standard knowledge]**; whether it bites here **[Hypothesis]**; see 07 D1.

## 8.3 Reliability mechanisms that exist (all [Verified])

| Mechanism | Where | Protects | Limit |
|---|---|---|---|
| `retry_async` (3 attempts; waits 1 s, 2 s; no jitter) | `services/llm_retry.py` | transient 429/5xx, malformed JSON | retries *every* exception (incl. `KeyError` for a missing API key); no breaker; lockstep; no deadline |
| Used at: resume, JD, question generation, report | respective modules | | **Not** used by `code_judge.review_code` or the memory-notes call |
| Pydantic + `validate_report()` | models, `report_generator.py` L41–58 | structurally wrong output | no range check (a score of 99 passes), no per-question validation (exercise 2.1) |
| Report status `pending → ready/failed`, retry endpoint | `report_store.py`, `report_pipeline.py`, `session.py` | UI never polls forever on a *reported* failure | `pending` has no staleness timeout (07 D3); TTL 86 400 s |
| Input limits: 5 MiB file, 30 000 resume chars, 15 000 JD chars, extension allow-list | `prepare.py` L56–59 | memory/cost abuse | no per-IP rate limit, no auth |
| Redis TTLs: plan 14 400 s, session 14 400 s, report 86 400 s, status 86 400 s, memory 1 year | `setex` sites | unbounded growth | snapshot-only persistence (§8.0) |
| Debounced autosave (≤1 write/s, whole blob) | `session_state.py` L93–110 | transcript survives an agent crash | reconnect does not *use* it (07 C1); failure only logged |
| Follow-up cap in code + prompt | `session_state.py` L13, L90, L149 | runaway questioning | soft (07 C2) |
| Library retries and session close on repeated STT/LLM/TTS errors | LiveKit Agents | provider blips in the voice path | see §8.8 |
| Optional-dependency fallbacks (tracing, turn detector) | `tracing.py`, `agent.py` | missing optional piece does not crash | degradation is silent |
| CI | `.github/workflows/ci.yml` | regressions | no frontend tests, no live-API tests |

## 8.4 Failure modes by what the user sees

| Failure | Detection | User-visible result | Label |
|---|---|---|---|
| Gemini error in prepare (after 3 tries) | exception in `event_stream` | generic SSE `error` message (detail only in log) | [Verified] design; rate [Not measured] |
| Invalid/oversized upload or JD | explicit checks | friendly `error` event | [Verified] |
| Plan missing/expired at token time | Redis `get` → 404 | "Session not found. Please run /api/prepare first." (`session.py` L37–42) | [Verified] |
| Missing LiveKit env vars | `KeyError` | HTTP 500 (reproduced offline, exercise 2.5) | [Measured] |
| Agent not running / crashed in `load()` | no participant | "connecting / taking too long" overlay after 25 s | [Inference] from UI code |
| Report generation fails | status `failed` | failure + retry button | [Verified] |
| Backend restart mid-retry | none | stuck `pending` (simulated state only, exercise 2.7) | [Hypothesis] |
| Piston down | per-test exception | tests recorded as failed (07 D7) | [Verified] code; not observed |
| Redis down at start | `lifespan` ping | backend refuses to start | [Verified] |

## 8.5 Scaling analysis (reasoned, not measured)
**Stateless pieces:** backend keeps no per-session state in process (only `_retry_tasks`, `_version_cache`) → can run several workers behind a load balancer **[Inference]**.
**Likely bottlenecks, in my estimated order [Inference]:**
1. **Gemini quota/latency**: every flow is LLM-bound; no queue, limiter, or timeout.
2. **Agent capacity**: one process per live interview, with streaming STT/TTS/VAD.
3. **Redis durability** (snapshot-only) before throughput (payloads are KB–100 KB).
4. **Event-loop blocking** from resume parsing.
5. **Piston**: sequential per-test calls; concurrent submissions queue **[Hypothesis]**.
**Cost drivers [Inference]:** STT minutes, TTS characters, and LLM input tokens per turn (§8.7 shows the system prompt alone is resent every call). No cost numbers were collected; quote none.
**Scale-up path (suggestions, not current code):** durable DB for reports/memory; queue + worker for reports; auth and rate limits; Gemini concurrency limiter + jitter + breaker + timeouts; multiple agent workers; per-stage metrics.

### 8.5.1 Worked capacity example (REASONING ONLY — assumptions flagged)
Goal: "20 interviews at once, each about 30 minutes". Every input is either [Measured] from local data or an explicit **assumption** (A).

*Gemini requests.*
- Candidate turns per interview: median **17** (range 16–30) across the 14 stored text-mode runs **[Measured]**. Assumption (A1): one extra LLM request per question advance (the tool turn's second call, ≈16). → about **33 live LLM requests** per interview, plus 3 (prepare) + 1 (report) = **≈37**. Over 30 min: 1.2 per minute per interview; ×20 = **≈25 requests/min ≈ 0.4 req/s** steady **[Inference]**.
- Input volume: with the system prompt of §8.7 resent per call, one request per candidate turn gives 0.88–1.39 million characters per interview (**≈220 k–350 k tokens by ÷4**) **[Measured arithmetic on synthetic plan + stored transcripts]**; with A1's extra calls up to ~2× that **[Inference]**. ×20 ≈ **9–14 million input tokens per 30 min** (≈0.3–0.5 M/min). Whether implicit prompt caching reduces billed/quota-counted tokens here **[Not measured]**; check `usage_metadata` in Langfuse.
- Burst: if all 20 end together, 20 report calls of roughly 16 k–32 k input tokens each and up to 16 k output tokens; median stored report is 36 k chars (≈9 k tokens by ÷4) → ~180 k output tokens in the burst **[Inference]**.
- **Missing number:** your project's Gemini requests/min and tokens/min quota. **[Not measured]; look it up in the provider console.** Compare it with 25 req/min and 0.3–0.5 M tokens/min before trusting anything above.

*Agent CPU/RAM.*
- Measured on this 12-core dev machine, Silero VAD alone consumed **0.035–0.041 CPU-seconds per second of 16 kHz audio** (three runs; synthetic noise, 10 ms frames) **[Measured]**, i.e. about 4 % of one core per stream. One process that merely imports LiveKit + plugins + `agent.interviewer` and loads VAD had **≈209 MB RSS** (no turn-detector model, no sockets) **[Measured, lower bound]**.
- Unmeasured per job: STT/TTS websocket handling, resampling, end-of-turn model inference, Python overhead. Define `x` = total cores per job (**unknown**). With LiveKit's `load_threshold` 0.7, capacity ≈ `0.7 × cores / x`:

| x (assumed, not measured) | 4-core box | 8-core box |
|---|---|---|
| 0.05 | 56 | 112 |
| 0.10 | 28 | 56 |
| 0.25 | 11 | 22 |

Memory floor: 20 jobs × ≥209 MB ≈ **4.2 GB** before the turn-detector model **[Inference]**. The VAD-only number cannot be used as the per-job cost; it is only a floor.

*Redis.* Per interview: plan ≈22 kB + session state up to ≈122 kB + report 29–72 kB ≈ **≈0.1–0.22 MB** (plan size synthetic, §8.7) **[Inference]**. 1 000 retained interviews ≈ 0.1–0.22 GB; trivial for RAM, but see §8.0 for durability. The autosave rewrites the **whole** session blob each time (≤1 Hz per session): ≈122 kB/s worst case per active interview at the largest stored transcript size **[Inference]**.

*Piston.* A coding submission runs `tests` sequential executes (3–5 per the generation prompt) at up to 3 s each: worst case 9–15 s of silent waiting for the candidate when code loops forever **[Inference]**; typical execution time **[Not measured]**.

*Conclusion (reasoning):* the first number to obtain is the Gemini quota; the second is real per-job CPU/RSS with the turn detector loaded; the plan for obtaining them is exercise 3.4.

## 8.6 What I would measure first (and how)
| Question | Measurement | Tooling that exists |
|---|---|---|
| Per-stage voice latency (end-of-speech → first audio) | LiveKit `metrics_collected` events + spans | library emits them; nothing here stores them |
| Is preparation dominated by one call? | span durations per `track_generation` | Langfuse (optional) |
| Report success rate | count `ready` vs `failed`, and stale `pending` | none; add counter |
| Silent-turn frequency in voice | transcripts | `deterministic_checks` (text harness only) |
| Thinking-token share of live latency | compare TTFT with `thinking_budget=0` | none (live API) |
| Max concurrent interviews per node | ramp test with scripted audio | none (exercise 3.4) |
| Cap violations | transcript check | `evals/checks.py` |

## 8.7 Token and size arithmetic ([Measured] chars; tokens = chars ÷ 4, a crude [Standard knowledge] estimate)
Method: I imported the real prompt constants with the repo venv and ran `len()`; built **synthetic** `InterviewPlan` objects (field lengths are **my assumptions**: question 150 chars, intent 100, 4 ideal points × 90, 2 follow-ups, coding specs 400+200 chars) because stored eval results contain no plans and Redis plans expire after 4 h; and ran `json.dumps` over the 14 stored `evals/results/eval-*.json` transcripts/reports. Real plans will differ: measure `len(redis GET interview_plan:<id>)` on your own.

**Static prompts (chars / ≈tokens):** interviewer intro 254 / 64; behavioural rules 2 421 / 605; whole interviewer system prompt with an *empty* plan **2 696** (exact); report template 1 259 / 315; code review 871 / 218; resume 916 / 229; JD 869 / 217; memory notes 634 / 158.

**Interviewer system prompt = rules + full plan, `indent=2` (`interviewer.py` L48–51, `prompts.py` L63–71), resent on every LLM call:**
| Synthetic plan | plan JSON | system prompt | ≈tokens |
|---|---|---|---|
| 8 questions | 14 527 | 17 223 | 4.3 k |
| 16 questions | 25 973 | 28 669 | 7.2 k |
| 6 questions, 4 coding | 18 151 | 20 847 | 5.2 k |
- Marginal cost ≈ **1 430 chars per question** (synthetic). `indent=2` adds ≈18 % over compact JSON (25 973 vs 21 929 chars at 16 questions).
- The plan includes `ideal_answer_points` and coding test cases (07 C8): the answer key is a large share of every prompt.

**Stored transcripts and reports (14 runs, text-mode, simulated candidates):** transcript JSON (`indent=2`, as `REPORT_PROMPT` builds it) 26 043–100 859 chars (median 38 160); report JSON 29 409–71 775 (median 36 269). The largest report is ≈17.9 k tokens by ÷4, above the 16 384 ceiling, yet it parsed; so ÷4 under-counts characters per token for this JSON **[Inference]** (also 07 A3). Report-call *input* ≈ plan (26 k) + transcript + 1.3 k: ≈65 k chars median **[Inference]**.

**Resend arithmetic (one LLM call per candidate turn; ignores tool-result calls and tool JSON):**
| Run | calls | total input chars across calls | system-prompt share |
|---|---|---|---|
| median-size run (`strong-9c2ee6a6`) | 19 | 881 194 (≈220 k tokens) | 62 % |
| largest (`prompt_injector-d292ba4d`) | 19 | 1 387 291 (≈347 k tokens) | 39 % |
Caveat: simulated-candidate answers average 415–4 358 chars; real spoken answers differ **[Not measured]**.

**Redis values (synthetic 16-question plan):** plan 21 929 B; empty session state 22 071 B; session state with the largest stored transcript 121 774 B (rewritten in full per autosave).

## 8.8 Failure-mode catalogue per dependency
Legend: "code does" = what our code or the library does on that failure.

### Gemini (backend call sites + live LLM)
| Call site | temp | max out | thinking | `retry_async` |
|---|---|---|---|---|
| resume analysis | 0.1 | 4 096 | 0 | yes |
| JD analysis | 0.1 | 4 096 | 0 | yes |
| question generation | 0.4 | 16 384 coding / 8 192 | −1 coding / 0 | yes |
| report | 0.2 | 16 384 | 0 | yes |
| code review | 0.2 | 4 096 | 0 | **no** |
| memory notes | 0.3 | 1 024 | 0 | **no** (caught in `report_pipeline` L149–164; report stays `ready`, memory not updated) |
| live agent LLM | 0.7 | not set | not set | LiveKit: 3 retries, then session close after >3 unrecoverable |

| Failure | Code does | Result |
|---|---|---|
| 429/5xx/network error | `retry_async` ×3 (1 s, 2 s) | recovers if outage <≈3 s of waiting; stored runs show a 503 outage outlasting it (07 B6) |
| Malformed/truncated JSON | `json.loads`/Pydantic raises → retried | each retry is a full paid call |
| Empty/blocked response (`response.text` is `None`) | `.strip()`/`_clean_json` raises `AttributeError` → retried **[Inference]** | same as above |
| Missing `GOOGLE_API_KEY` | `KeyError` inside the retried function | 3 attempts, 3 s of waiting, then failure **[Inference]** |
| Hung request | no timeout in our code; SDK default `None` | can wait indefinitely **[Inference]** |
| Quota exhaustion | indistinguishable from other errors; lockstep retries | amplifies load (07 D2) |
| Live-path failure | library retries then closes session on repeated errors | shutdown callback → report from partial transcript **[Library source + Inference]** |

### Soniox (STT)
| Failure | Code does |
|---|---|
| Connection/auth error | library: up to 3 retries, 10 s timeout each; errors counted, reset on a transcript; >3 consecutive unrecoverable → session closes |
| Pinned old model (`stt-rt-v4`) retired | nothing in our code; would surface as repeated STT errors **[Hypothesis]** |
| Wrong language/accent | `language_hints=["en"]` only; transcription quality **[Not measured]** |
Candidate-visible effect: the agent stops hearing; no custom message exists.

### Cartesia (TTS)
| Failure | Code does |
|---|---|
| API error / bad voice id | library retries ×3, then closes the session after >3 consecutive unrecoverable errors; opening line failure is only logged and the session continues silently (`interviewer.py` L67–68) |
| Interrupted speech | transcript may keep the full text that was not fully spoken (02c §2c.4) |

### LiveKit (rooms, dispatch, transport)
| Failure | Code does |
|---|---|
| Missing env vars at token time | `KeyError` → HTTP 500 (07 C19) |
| Token minted locally | no network call at token time **[Verified]**; Cloud outage shows up at browser connect |
| Agent not registered / not dispatched | nothing; UI overlay after 25 s |
| Plan missing in job metadata | agent logs and returns (`agent.py` L174–197) |
| Participant drops | session closes; shutdown callback runs the report **only if `ended_at` unset** (`agent.py` L251–257) |
| `publish_data` failure | logged only (`interviewer.py` L36–41) |

### Redis
| Failure | Code does |
|---|---|
| Down at backend start | startup fails (fail fast) |
| Down at runtime, backend handler | unhandled exception → HTTP 500, except the memory path which is wrapped |
| Down at agent start | `load()` raises non-`ValueError` → job fails (not caught) |
| Down during interview | autosave logs; state not saved until next entry |
| Down at report time | `set_status` is inside the `try`; failure to set `failed` is itself logged → report lost, transcript maybe unsaved |
| Crash/restart | snapshot-only persistence (§8.0): writes since last snapshot lost; volume may be dropped |
| TTL expiry | plan gone after 4 h → token 404; report/status after 24 h; no user warning (07 D10) |

### Piston
| Failure | Code does |
|---|---|
| Container down | each test becomes `passed=False`, stderr `execution error: …`; the LLM still reviews against "0 tests passed" (07 D7) |
| Language not installed | `_ensure_language_version` tries to install a package inside the same request; the 30 s HTTP timeout may be shorter than an install **[Hypothesis]** |
| Infinite loop | killed at 3 000 ms; `signal` recorded; test fails |
| Slow tests | sequential calls; the candidate hears nothing meanwhile **[Inference]** |
| Wrong answer vs infra error | indistinguishable in the score |

## 8.9 Interview-ready summary
"Reliability here is mostly *containment*: validate LLM output, retry with short backoff, record explicit status, cap sizes, expire state. Missing: timeouts, jitter, durability beyond Redis snapshots, auth, rate limits and voice metrics. I measured prompt and payload sizes and one component's CPU, and I reasoned about capacity with every assumption labelled, but I have **no measured end-to-end load numbers**, so I do not claim capacity."
