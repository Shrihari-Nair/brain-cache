# 02 — Technology Deep Dives (index and method)

> The technology chapters are split by area so each can go deep. Every technology section follows **the same eight-part structure** so you can compare them and so nothing important is skipped.

## 2.1 The eight-part structure used in every technology section

| Part | Question it answers |
|---|---|
| **A. What is it?** | A plain-English definition |
| **B. Why does it exist?** | The problem it solves; what is hard without it |
| **C. How does it work internally?** | The mechanism, step by step — and which parts the *library* does vs which parts *our code* does |
| **D. How is it used in this codebase?** | Real files, classes, functions, config, call sites |
| **E. Example** | A small snippet; **illustrative** vs **actual** is always stated |
| **F. Alternatives** | Other realistic ways to solve the same problem |
| **G. Trade-offs** | Benefits, limits, performance, complexity, failure modes |
| **H. Interview connection** | Likely questions and follow-ups (full answers in [11](11-interview-preparation.md)) |

## 2.2 The chapters

| Chapter | Technologies |
|---|---|
| [02a — Backend and data](02a-backend-and-data-technologies.md) | Python `asyncio`; FastAPI + ASGI + Uvicorn; Pydantic; Redis (`redis.asyncio`); Server-Sent Events; resume parsing (`pypdf`, `python-docx`); HTTP client (`aiohttp`) |
| [02b — AI and LLM](02b-ai-and-llm-deep-dive.md) | Gemini API (`google-genai`); prompts and prompt assembly; structured output & validation; thinking budget; tool calling; workflow vs agent classification; conversation state; memory; safety/privacy; LLM evaluation; verification that RAG/embeddings are absent |
| [02c — Real-time voice and LiveKit](02c-real-time-voice-and-livekit.md) | WebRTC and SFUs; LiveKit rooms/tokens/dispatch/workers/data channels; the STT→LLM→TTS pipeline; VAD; turn detection; interruptions; latency |
| [02d — Frontend](02d-frontend-technologies.md) | Next.js App Router; React hooks; TypeScript; Tailwind; LiveKit React components; Monaco; browser media APIs; print CSS |
| [02e — Infrastructure and tooling](02e-infra-and-tooling.md) | Docker & Compose; environment configuration; Piston sandbox; Langfuse & OpenTelemetry; pytest; GitHub Actions; uv/npm |

## 2.3 A note on "what the library does vs what our code does"
Most technologies here are **libraries/services we configure**, not things we implement. Interview answers are strongest when they separate these clearly. Where it matters (for example LiveKit in [02c §2c.1](02c-real-time-voice-and-livekit.md#2c1-webrtc-sfus-and-livekit)) the "How it works" part ends with a short **"Responsibility split"** table; elsewhere the split is stated in prose. Not every section uses all eight parts in full: small topics merge parts (for example "A-B" or "F-G").

## 2.4 Corrections that apply across chapters
- **Redis persistence:** the app's Redis in `docker-compose.yml` has no explicit persistence settings, so the `redis:7-alpine` image defaults apply: RDB snapshots (`save 3600 1 300 100 60 10000`), AOF off, anonymous `/data` volume. Older wording such as "restart loses everything" is wrong; recent writes can be lost and the data goes away if the volume is removed. Details and the measurement: [02a](02a-backend-and-data-technologies.md) (Redis section), [03 §3.5](03-system-architecture.md), [07 C4](07-engineering-challenges.md).
- **Frontend port:** compose publishes the frontend on host port 3002; the README assumes 3000 ([07 C28](07-engineering-challenges.md)).
