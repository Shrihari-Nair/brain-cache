# 01 — Foundations

This chapter teaches only the prerequisite ideas you need to follow the rest of the handbook, building from the basics upward. Each section answers four questions: what it means, what problem it solves, how it works, and where it shows up in MockMind. If you already know a topic, skip it; later chapters link back here when they rely on a concept.

Essential sections are marked **(essential)**; the rest are helpful background.

## 1. Clients, servers, HTTP and APIs (essential)

**Simple meaning.** A server is a program that waits for requests; a client sends them. On the web, the language is HTTP. A request has a method (`GET` to read, `POST` to send), a path, headers and an optional body. A response has a status code and a body. An API endpoint is just a method plus a path that programs, not people, call, such as `POST /api/token`.

**Status codes** tell the caller what happened, and knowing who produced them is a core debugging skill: 200 means OK; 404 means "not found"; 422 means the framework rejected malformed input before your code ran; 502 means an upstream service failed. In MockMind, 404 on the report endpoint can legitimately mean "not ready yet, keep asking", while 502 means "generation failed".

**JSON** is the text format for structured data that these calls carry. A **multipart** request carries a file plus fields, which is how the resume is uploaded.

**CORS** is a browser-only safety rule: a web page may read responses from a different origin (host and port) only if that server allows it. The frontend and backend run on different ports, so the backend must list the frontend's origin. `curl` ignores CORS, so "works in curl, fails in the browser" is a classic symptom.

**Why it matters here.** The backend is a FastAPI service exposing a handful of endpoints (prepare, token, report, retry, memory, health). Where to look: `mockmind/backend/routers/`.

## 2. Streaming: polling, SSE, WebSocket, WebRTC (essential)

Sometimes one request and one response is not enough. There are four common styles:

| Style | Idea | In MockMind |
|---|---|---|
| Polling | Client asks again and again | The report page asks every few seconds until the report exists |
| Server-Sent Events (SSE) | Server pushes a stream of messages over one open HTTP response | Preparation progress |
| WebSocket | Persistent two-way channel | Used inside LiveKit, not by our code |
| WebRTC | Browser real-time audio, video and data, low latency | The live interview |

**SSE in practice.** Preparation takes many seconds (three LLM calls). Instead of a blank page, the server keeps the response open and writes lines such as `data: {...}` as each stage completes. The browser cannot use its built-in `EventSource` here because that only supports `GET` without a body, and we must `POST` a file; so the frontend reads the response stream manually.

**The streaming gotcha.** Once the server has sent the first bytes, the status code (200) is already gone. If something fails halfway, the only way to tell the client is a message inside the stream. So "HTTP 200" does not mean "it worked" for a streamed endpoint. This shapes how errors are modelled in the preparation flow ([03-how-it-works.md](03-how-it-works.md)).

## 3. Async programming (essential)

**Simple meaning.** Most of what a web backend does is wait: for an LLM, for Redis, for the network. `async`/`await` lets one thread juggle many such waits. When a task hits `await`, it pauses and the **event loop** runs something else until the awaited operation completes. This is cooperative concurrency on a single thread, ideal for I/O-bound work.

**The rule that matters.** A task only yields at an `await`. A slow *blocking* call (heavy computation, a synchronous file parser, `time.sleep`) never yields, so everything else on that process, including health checks and other users, stalls until it returns.

**Example in this repo.** Resume text extraction uses a plain synchronous PDF parser called directly from an async endpoint, so while one large PDF is parsed, other requests on that backend process could wait. That it *can* block is standard knowledge; how long it takes is **[Not measured]**. By contrast, the sleep inside LLM retry backoff is awaited, so other requests proceed during it.

**Background tasks.** `asyncio.create_task` starts work without waiting for it. Python only holds a weak reference to such tasks, so a task nobody references can be garbage-collected mid-run. The report-retry code keeps tasks in a module-level set for exactly this reason.

**Processes.** The LiveKit worker runs each live interview in its own OS process by default, so a stall in one interview does not freeze another. Where to look: `mockmind/backend/routers/session.py` (retry tasks).

## 4. Types, schemas and validation (essential)

**Simple meaning.** A schema describes the shape of data: which fields exist and what type each has. Validation checks real data against that shape. Python type hints alone are not enforced at runtime; **Pydantic** makes them enforced.

**Problem solved.** An LLM returns text. Even in JSON mode, the JSON may have a missing field, a wrong type, or an enum value you never allowed. If that data flows into Redis and later into the voice agent, it crashes far from where it went wrong. Validating at the boundary makes the failure immediate and local.

**A real incident [Verified].** The question generator once returned `"category": "coding"`, which the `QuestionCategory` enum did not include. Validation raised an error on roughly every other real call; the fix is in commit `b2ba44e`. Two Pydantic defaults surprise newcomers: unknown fields are ignored, and `"4"` is coerced to `4`.

**Why it matters here.** Every LLM output in the backend (candidate profile, job profile, interview plan, report, code review, memory update) is parsed into a Pydantic model. The same models are shared with the agent. Where to look: `mockmind/backend/models/`.

## 5. Redis, TTL and shared state (essential)

**Simple meaning.** Redis is a fast key-value store held in memory. You save a string under a key and read it back. A **TTL** (time to live) tells Redis to delete the key automatically after a number of seconds.

**Problem solved.** MockMind has two separate processes (the API and the voice agent) that need to exchange data: the plan, the transcript, the report. Rather than calling each other, they both read and write Redis keys. TTLs mean abandoned sessions clean themselves up with no cleanup job.

**How it works here.** Models are serialised to JSON text on write and parsed on read. The interview plan lives for four hours; reports for a day; candidate memory for a year under a hash of the email, so raw emails never appear in keys. If the plan key expires, joining returns 404.

**Limits to know.** Each single Redis command is atomic, but a read-modify-write sequence is not, so two writers could overwrite each other. Also, the Compose Redis uses the image's default snapshots, not append-only logging, so recent writes can be lost if it crashes; it is not "no persistence", but it is not durable either. And there is no SQL database in this application: no tables, no migrations. See [02-architecture.md](02-architecture.md) for the shared-state contract.

## 6. Tokens, JWTs and trust (essential)

**Simple meaning.** A JWT is a signed token in three parts: header, payload, signature. Anyone can read the payload; nobody can change it without invalidating the signature, which requires a secret only the server holds.

**How it works here.** To let the browser join a LiveKit room, the backend mints a JWT that says who the participant is, which single room they may join, what they may publish, an expiry of two hours, and a room configuration asking LiveKit to dispatch the agent named `mockmind-interviewer`. The signing secret never leaves the backend.

**What it does not do.** Nothing authenticates the person asking for the token: anyone who knows a valid session id can get one. A random unguessable session id acts as a **capability URL**: possessing it is the credential. This is an acceptable trade-off for a demo and a real limitation for anything public; [05-engineering-decisions.md](05-engineering-decisions.md) treats it as a design limitation.

## 7. LLM basics (essential)

**Simple meaning.** A large language model predicts the next **token** (a chunk of text, roughly three quarters of a word) over and over. You send a **prompt**; a **system prompt** gives standing instructions.

**Knobs that matter here.**

- **Temperature**: randomness. Lower is steadier.
- **Max output tokens**: a hard cap. If the answer is longer, it is cut off mid-sentence, which for JSON means an unparseable result.
- **Context window**: total tokens the model can consider.
- **Thinking tokens**: some models (Gemini 2.5) spend hidden reasoning tokens before answering. They **count against the output budget**, so a large amount of thinking can leave too little room for the visible JSON. The code reviewer therefore sets the thinking budget to zero, and the report truncation problem recorded in `mockmind/evals/REPORT.md` is the same mechanism.
- **Hallucination**: fluent but false output. This is why code review is grounded in real execution output rather than the model's guess.
- **Prompt injection**: instructions hidden in data the model reads ("ignore previous instructions"). Any text from a candidate, including a resume and stored memory notes, is untrusted data, not instructions.

**Structured output.** Asking for JSON (a response MIME type or schema) makes syntax mostly reliable. It does not guarantee your schema or truthfulness, hence section 4. So the code defends in layers: ask for JSON, strip stray markdown fences, validate with Pydantic, retry on failure.

**Retry with backoff.** LLM APIs fail transiently (rate limits, overload). Retrying after a growing delay (exponential backoff) usually succeeds without hammering the service. Where to look: `retry_async` in `mockmind/backend/services/llm_retry.py`. Retrying is only safe when repeating the call is harmless; the term for that is idempotent.

## 8. Tool calling, workflows and agents (essential)

**Tool calling.** Instead of replying with text, the model can reply "please run function X with these arguments". Your code runs it and feeds the result back. This is how a model affects the world.

**Workflow vs agent.**

| | Workflow | Agent |
|---|---|---|
| Who chooses the next step | Your code | The model, in a loop |
| Predictability | High | Lower |
| Good for | Known sequences | Open-ended tasks |

MockMind uses both, which is the main thing to understand. **Preparation is a workflow**: resume analysis, job analysis, question generation, each a single LLM call in a fixed order. **The live interviewer is a constrained agent**: the model chooses between a follow-up and a tool such as `move_to_next_question` or `end_interview`, but the question sequence itself is controlled by code. There is no retrieval (RAG), no embeddings, and no multiple cooperating agents.

**Soft versus hard enforcement.** If a rule must always hold, put it in code. If you can only tell the model, it is a request. The follow-up cap (two) is soft: a counter and a note in the prompt each turn, plus an eval check that detects violations, but the model can still ignore it. This distinction recurs throughout the handbook.

## 9. Real-time voice (essential)

A voice interview is a pipeline, and each stage adds delay:

1. **Capture and transport.** The microphone is sampled, compressed (Opus) and sent in small packets over **WebRTC**. A media server (an **SFU**, here LiveKit Cloud) relays streams between participants. The browser and the agent both join a **room**.
2. **VAD (voice activity detection)** notices when someone is speaking.
3. **Turn detection** decides whether they have finished, or merely paused. This is harder than it sounds, since people pause mid-thought.
4. **STT (speech-to-text)** transcribes the speech.
5. **LLM** decides the reply.
6. **TTS (text-to-speech)** speaks it.

This STT-LLM-TTS chain is called a **cascaded** voice agent; the alternative is a single speech-to-speech model. Cascading is easier to inspect and swap parts of, at the cost of added latency per stage. **Barge-in** is the candidate interrupting the agent mid-sentence; the system must stop speaking and listen.

**LiveKit vocabulary.** A **worker** is the long-running agent program registered with LiveKit. A **job** is one interview assigned to it. **Dispatch** is how LiveKit decides which agent joins a room; here the dispatch is explicit and requested through the candidate's token. A **data channel** carries small non-audio messages (for example, opening the code editor). Latency per stage is **[Not measured]** in this repository, so do not quote numbers.

## 10. Containers and configuration

**Containers.** A container packages a program with its dependencies so it runs the same anywhere. **Docker Compose** starts several together. Containers on the Compose network reach each other by service name (`redis`, `piston`), not `localhost`.

**The `localhost` trap.** Inside a container, `localhost` means that container. So the same Redis URL must differ between "running on my laptop" and "running in Compose", and the Compose file overrides it. The browser, however, always runs on your host, so frontend URLs keep `localhost`.

**Configuration.** Secrets and endpoints come from environment variables in a git-ignored `.env`, with a committed example file. Variables prefixed `NEXT_PUBLIC_` are baked into the browser code at build time, which means they are public by design and must never hold secrets. Piston runs privileged in Compose so it can sandbox untrusted code, a trade-off acceptable locally and notable for any wider deployment.

## 11. Testing a non-deterministic system

**Unit tests** check one function; **integration tests** check parts together; **fakes** stand in for dependencies. In MockMind the test suite replaces Redis with an in-memory fake and stubs LLM calls, so tests are fast and free. That proves your code handles a given model answer correctly. It cannot prove what a real model will do, which is why a separate **eval harness** runs simulated candidates against real models and applies deterministic checks (for example, "did the interviewer exceed two follow-ups"). Evals cost money and are non-deterministic, so here they are run manually, not on every push. The full strategy is in [06-quality-and-reliability.md](06-quality-and-reliability.md).

## 12. Reliability vocabulary

These terms are used throughout; each is developed where it appears.

| Term | Meaning | Where you will meet it |
|---|---|---|
| Timeout | Stop waiting after a limit | Code execution, agent join |
| Graceful degradation | An optional part fails, the core continues | Candidate memory |
| Fail fast | Stop loudly on an essential failure | Redis check at backend start |
| State machine | Named states with allowed transitions | Report status: pending, ready, failed |
| Checkpointing | Save progress as you go | Transcript autosave |
| Idempotency | Repeating is harmless | Report retry while pending |
| Defence in depth | Several layers for one rule | The follow-up cap |

## Self-check

1. Why can a streamed endpoint return HTTP 200 and still have failed?
2. What stalls if you call a blocking function inside an `async` handler?
3. Why do the backend and agent share Redis instead of calling each other?
4. What does JSON mode not guarantee?
5. Why can hidden thinking tokens break a JSON response?
6. How is a workflow different from an agent, and which is the live interviewer?

Short answers: (1) the status is sent before the work finishes, so later errors must travel inside the stream; (2) the entire event loop on that process, so every other request waits; (3) they are separate processes, and Redis is a simple shared store with automatic expiry; (4) your schema, valid enum values, and truthfulness; (5) they consume the output token budget, truncating the visible JSON; (6) a workflow's sequence is chosen by code, an agent's by the model; the interviewer is a constrained agent inside a code-controlled question sequence.

## Recap

You now have the working vocabulary: HTTP and SSE for the web half, async for concurrency, Pydantic for trusting nothing the LLM says, Redis with TTLs as the glue, JWTs for room access, LLM knobs and tool calling for intelligence, and the STT-LLM-TTS chain for voice. Next, [02-architecture.md](02-architecture.md) shows how these parts are assembled into components, and why the boundaries sit where they do.
