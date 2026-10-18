# 02d — Frontend Technologies

> **Scope:** Next.js (App Router), React hooks and state, TypeScript, Tailwind CSS, the browser media/streaming APIs, `@livekit/components-react`, Monaco editor, print CSS.
> **Versions [Verified]** (`mockmind/frontend/package.json`): `next` **14.2.0** (exact), `react`/`react-dom` `^18`, `typescript` `^5`, `tailwindcss` `^3.4.0`, `@livekit/components-react` `^2.6.0`, `livekit-client` `^2.5.0`, `@monaco-editor/react` `^4.7.0`, `lucide-react` `^1.48.0`. `package-lock.json` pins exact installs.
> **Prerequisites:** basic HTML/JavaScript; [01 §1.11](01-fundamentals.md#111-audio-real-time-and-voice-fundamentals) for the WebRTC terms; [02c](02c-real-time-voice-and-livekit.md) for what the agent does on the other end of the room.
> **File inventory [Verified]:** every file under `mockmind/frontend/` (excluding `node_modules`, `.next`) is covered in section 2d.11 below.
> **Not present [Verified]:** a frontend test framework, ESLint/Prettier configuration (the `lint` script `next lint` exists but there is no ESLint setup), a global state library.

---

## 2d.1 Next.js App Router

### A. What is it?
A React framework providing file-based routing, a build pipeline, server rendering and bundling. In the **App Router**, folders under `app/` define routes; `page.tsx` is the page; `[param]` folders are dynamic segments.

### B. Why does it exist?
Plain React gives you components but not routing, build optimization, code-splitting or a server runtime. Next.js bundles these.

### C. How it works
At build time Next compiles each route. Components are **Server Components by default** (rendered on the server, no browser APIs/hooks); a file starting with `'use client'` becomes a **Client Component** (hydrated in the browser, can use `useState`, effects, `window`, `navigator`). Environment variables prefixed `NEXT_PUBLIC_` are **inlined into the browser bundle at build time**.

**Worked example: what "server" vs "client" means for `/`.** `app/page.tsx` has no `'use client'` and no hooks, so Next renders it to HTML at build/request time on the Node server and sends finished markup (`next/link` handles the navigation to `/setup`). `app/setup/page.tsx` starts with `'use client'`: the server still pre-renders its first HTML, but the browser then downloads its JavaScript and **hydrates** it (attaches event handlers and state) so `useState`/`onClick` work. The interview page is a client component *and* depends on browser-only APIs (`navigator.mediaDevices`), which is why it must be one.
**`reactStrictMode: true` consequence [Standard knowledge, Verified setting]:** in development React deliberately mounts, unmounts and re-mounts every component once, so effects run twice. On the report page the initial `fetchReport()` effect, and on the interview page the mic/token effect, therefore run twice in `next dev`. The interview effect's `cancelled` flag exists partly for this: the first run's result is discarded. Production builds do not double-run.

### D. Used here
| Route | File | Component type |
|---|---|---|
| `/` | `mockmind/frontend/app/page.tsx` (`LandingPage` L131) | Server (no hooks) |
| `/setup` | `app/setup/page.tsx` (`SetupPage` L91) | Client |
| `/interview/[sessionId]` | `app/interview/[sessionId]/page.tsx` (`InterviewPage` L36) | Client |
| `/report/[sessionId]` | `app/report/[sessionId]/page.tsx` (`ReportPage` L43) | Client |
Root layout `app/layout.tsx` loads two Google fonts via `next/font/google` (CSS variables `--font-display`, `--font-sans`), sets metadata, and imports `@livekit/components-styles`. `next.config.js` only sets `reactStrictMode: true`. The Dockerfile is a three-stage build (deps → builder with `NEXT_PUBLIC_*` as **build args** → runner) (`mockmind/frontend/Dockerfile`).
**Possible fresh-clone build break [Inference; image not built here]:** the runner stage does `COPY --from=builder /app/public ./public`, but `mockmind/frontend/public/` is an **empty directory** locally (`ls -la` shows no files) and git does not track empty directories (`git ls-files frontend` lists no `public/` entry). On a fresh clone the builder stage would have no `/app/public`, and Docker's `COPY` of a missing source fails the build. Fix would be a placeholder file in `public/` or removing that line; outside the scope of this documentation pass.
**Where the variables come from in Compose [Verified]:** `docker-compose.yml` passes `NEXT_PUBLIC_LIVEKIT_URL: ${LIVEKIT_URL}` and a hard-coded `NEXT_PUBLIC_BACKEND_URL: http://localhost:8000` as build args and publishes the container on host port **3002** (`"3002:3000"`), whereas the README and the backend's default `CORS_ORIGINS` assume port 3000 ([07](07-engineering-challenges.md) C28). `NEXT_PUBLIC_LIVEKIT_URL` is baked into the bundle but no frontend source file reads it (the LiveKit URL comes back from `/api/token` as `ws_url`); only the Dockerfile and compose mention it **[Verified by grep]**.

### E. Example (actual, abbreviated): reading the dynamic route parameter
```tsx
const { sessionId } = useParams() as { sessionId: string };   // interview/[sessionId]/page.tsx
```

### F. Alternatives
Vite + React Router (SPA), Remix, Astro, plain HTML + HTMX. For an app with 4 pages and no SEO needs, a SPA would also work; Next gives routing and build tooling.

### G. Trade-offs
+ Conventions, optimized builds. − Server/Client component boundary confusion; build-time env baking means changing `NEXT_PUBLIC_BACKEND_URL` requires a rebuild **[Verified in Dockerfile args]**. − The Docker image uses `node:18-alpine` while CI uses Node 20 **[Verified]**.

### H. Interview connection
"Which pages are server vs client components and why?" "Why are `NEXT_PUBLIC_` variables build args?" "What's the difference between SSR and hydration?"

---

## 2d.2 React: components, state, effects, custom hooks

### A–B. What/Why
UIs are described as functions of state. When state changes, React re-renders the affected components.

### C. How it works
- `useState` returns `[value, setter]`; calling the setter schedules a re-render.
- `useEffect(fn, deps)` runs `fn` after render when `deps` change; the returned function is **cleanup**.
- `useRef` holds a mutable value without triggering renders; `useCallback` stabilizes function identity.
- A **custom hook** is a function using hooks to share logic.

### D. Used here
| Pattern | Source |
|---|---|
| Streaming progress hook | `mockmind/frontend/hooks/usePreparation.ts` → `usePreparation()` (L13+) |
| Token + microphone effect with cancellation flag | `interview/[sessionId]/page.tsx` `InterviewPage` effect (L43–58) |
| 25 s agent-join timer with cleanup | same file effect L363–371 (`setTimeout(..., 25_000)` at L368) |
| Report polling with `setTimeout` chain and a 30-attempt cap | `report/[sessionId]/page.tsx` effects (L69, L74) |
| Reusable radio-group control | `app/setup/page.tsx` → `Segmented<T>` (L37) |
| Derived state instead of extra state | `canSubmit` in `SetupPage` (`resumeFile && !fileError && JD length 51..15 000 && !isRunning`) and `connectionLost` in `InterviewRoomUI` are plain expressions recomputed on each render |
| Latest-callback safety | `handleSubmitCode` uses `useCallback` with `[challenge, sendCodeSubmission]` (page L339-350) |
| Unused hook | `hooks/useInterviewSession.ts` (L6) is exported but not imported anywhere **[Verified by grep]** |

### E. Example — the cancellation pattern (actual, simplified)
```tsx
useEffect(() => {
  let cancelled = false;
  checkMicrophone().then((r) => { if (cancelled) return; /* use r */ });
  return () => { cancelled = true; };      // prevents setting state after unmount / re-run
}, [sessionId, attempt]);
```
**Why:** async work can finish after the component unmounted; without the flag it would update state for a view that no longer exists.

**Worked example: why the report page has two effects.** Effect 1 (L69) calls `fetchReport()` once. `fetchReport` ends with `setPollCount(n+1)`. Effect 2 (L74) depends on `[polling, pollCount, fetchReport]`, so each completed fetch changes `pollCount`, which re-runs effect 2, which schedules the next `setTimeout(fetchReport, 4000)` and returns a cleanup that cancels the pending timer. This is a **timer chain driven by state** rather than `setInterval`: a slow response can never overlap with the next request. When `pollCount >= 30` effect 2 sets `polling=false` and the page switches to the "taking longer than expected" view.

### F–G. Alternatives and trade-offs
Global state (Redux/Zustand) — unnecessary here since the only cross-page state is the session id in the URL. Data-fetching libraries (SWR/React Query) would replace the hand-written polling. No frontend tests exist, so UI logic regressions are caught only manually or by `tsc`/`next build` in CI **[Verified]**.

---

## 2d.3 TypeScript

TypeScript adds static types, checked at build time by `tsc` (`tsconfig.json`: `strict: true`, `noEmit: true`) and erased at runtime. CI runs `npx tsc --noEmit` (`.github/workflows/ci.yml`), which I ran and observed exit code 0 **[Measured]**.
**Contract mirroring:** `mockmind/frontend/types/interview.ts` hand-copies backend shapes (`PrepProgressEvent`, `SessionTokenData`, `EvaluationReport`, …). **Verified drift:** TS `TranscriptEntry.speaker` lacks `'system'` and `question_index`; TS `InterviewQuestion` has 5 fields vs the Python model's richer set. Types are compile-time only: a mismatch with the real JSON isn't detected at runtime.
**More drift [Verified]:** `types/interview.ts` declares `TranscriptEntry` and `InterviewQuestion`, but nothing in the frontend imports them (grep), and `EvaluationReport`/`QuestionFeedback` match the JSON schema in the backend's `REPORT_PROMPT` (checked field by field). `validate_report` guarantees only `overall_score`, `hiring_recommendation`, `category_scores` and defaults four others, so the fields *inside* each `question_feedback` entry are trusted from the LLM output without validation **[Verified by reading; consequence for the page is an Inference]**. `SessionTokenData` matches the five keys `get_token()` returns (`token, ws_url, room_name, session_id, identity`). `CodingChallenge` lives in `components/CodeEditorPanel.tsx`, not in `types/`.
**Discriminated union example (actual):** `ReportResult = {status:'ready',report} | {status:'pending'} | {status:'failed'}` in `lib/api.ts` (L31+).

---

## 2d.4 Browser streaming: `fetch` + `ReadableStream` for SSE over POST

### A–C. What/why/how
`EventSource` can only issue GET without a body, so `usePreparation()` uses `fetch(..., {method:'POST', body: formData})` and reads `response.body.getReader()`. Network chunks do **not** align with SSE event boundaries, so the code keeps a **buffer**: it appends decoded text, splits on `\n`, keeps the last (possibly incomplete) piece for the next read, and parses lines starting with `data: `.

### D. Source
`mockmind/frontend/hooks/usePreparation.ts` → `usePreparation()` (reader loop; buffer handling; `AbortController`; sets `events`, `currentProgress`, `sessionInfo`, `error`).

### E. Worked example (illustrative)
```text
chunk 1: 'data: {"stage":"parsing","progress":5}\n\ndata: {"stage":"resu'
→ complete line parsed; 'data: {"stage":"resu' kept in buffer
chunk 2: 'me_analysis","progress":45}\n\n'
→ buffer + chunk = complete line → parsed
```
Without the buffer, `JSON.parse` would throw on the half line (the code also wraps parsing in `try/catch` and skips malformed lines).

### F. Alternatives
`EventSource` (GET only, cannot send a file), WebSockets (heavier; no need for two-way), polling a job id (needs a job table, which this app does not have), or `fetch` + `eventsource-parser`-style libraries (the hand-written loop is ~25 lines).

### G. Trade-offs
**Edge cases in the parser [Verified by reading]:** it splits on `\n` and requires the line prefix `data: `; the server's `_sse()` sends `data: {json}\n\n`, so blank separator lines are skipped. A final event that arrives without a trailing newline would stay in `buffer` and never be parsed (the server always ends events with a newline, so this does not occur here). If the server dies mid-stream the loop ends normally with `done`, `isRunning` becomes false, and no `complete` or `error` event was received: the UI shows neither "Interview ready" nor an error (the "Prepare" button simply becomes enabled again). Non-2xx responses (for example a 422 from FastAPI form validation) throw "Preparation request failed".
No resume if the connection drops mid-preparation (SSE here has no event ids); no retry on the client; error text is shown only if the server sends an `error` event or the request fails.

---

## 2d.5 Browser media: `getUserMedia` and the microphone pre-flight

`checkMicrophone()` (`interview/[sessionId]/page.tsx` L25–35) calls `navigator.mediaDevices.getUserMedia({audio:true})`, immediately stops the tracks, and returns `'granted' | 'denied' | 'unsupported'`. **Why:** a denied microphone otherwise results in an interview where the agent can't hear the user; failing early gives a clear message. The browser's own prompt is triggered by this call. Requires a secure context (HTTPS or localhost) **[Standard knowledge]**; the 'unsupported' branch in the code covers `navigator.mediaDevices` being undefined (insecure origin or very old browser), while *any* thrown error (permission denied, no device, device busy) is lumped into `'denied'`, so a missing microphone shows the "blocked" message.
**Worked example:** user opens `/interview/abc` on plain `http://192.168.1.5:3000` (not localhost): `navigator.mediaDevices` is undefined -> `'unsupported'` -> screen says to use HTTPS or localhost. Same user on `http://localhost:3000` with the mic blocked in site settings: `getUserMedia` rejects -> `'denied'` -> screen explains the lock icon; **Try again** increments `attempt`, re-running the effect.

---

## 2d.6 `@livekit/components-react` (the live room UI)

| Piece | Role | Source |
|---|---|---|
| `<LiveKitRoom token serverUrl connect audio video={false}>` | connects with the token, publishes the mic, provides context | `InterviewPage` render |
| `<RoomAudioRenderer />` | plays remote audio (the agent's voice) | same |
| `useVoiceAssistant()` | agent state (`speaking`/`listening`/`thinking`/…) | `InterviewRoomUI` (L316) |
| `useConnectionState()` | `Connected`/`Disconnected`… for the "Connection lost" overlay (`connectionLost`, L376) | same |
| `useDataChannel(topic, cb)` | receive/send data messages | L326, L337 |
| `useRoomContext()` | `room.disconnect()` on End Interview (`handleEndInterview`, L379–384) | same |
**Props and behaviour worth knowing [Verified]:** `audio={true}` makes the component request the mic and publish it on connect; `video={false}` avoids a camera prompt; `connect={true}` connects as soon as the component mounts with a token; no `onConnected/onDisconnected/onError` callbacks are passed. All hooks (`useVoiceAssistant`, `useDataChannel`, `useRoomContext`, `useConnectionState`) must be called inside `<LiveKitRoom>`, which is why `InterviewRoomUI` is a separate child component rather than inline in `InterviewPage`. `useDataChannel('code_submission')` (no callback) is used only for its `send` function; with a callback and a topic it filters incoming messages by topic.
**Orb UI:** `VoiceOrb` (L189-312) renders per-state gradients/animations from `ORB_CONFIG` (L134-187; four states `speaking | listening | thinking | disconnected`, each with label, sublabel, pill/dot classes, gradient, ring colour and two CSS animation names). Keyframes are injected through an inline `<style>` tag inside `VoiceOrb` rather than `globals.css`; three expanding rings with staggered delays (0, 0.5, 1.0 s) show only while active. It uses **inline styles** for sizes and gradients, which is the one place the app deviates from Tailwind-only styling.
**The 25 s timer, the mic gate and the connection-lost logic are explained with state tables in [02c §2c.11](02c-real-time-voice-and-livekit.md#2c11-agent-and-user-states-and-the-ui).**
**Failure UX added in code [Verified]:** pre-flight mic screen; "Connection error" with Try again; a 25 s "interviewer is taking too long" overlay; a "Connection lost" overlay with **Reconnect** (which does `window.location.reload()`) and **Get my report**.
**Limitation [Verified, behavior derived from code]:** Reconnect reloads the page → new token → new agent job whose `AgentSessionState.load()` reads only the plan, so the interview restarts from question 1 and its autosave overwrites `session_state:{id}`. See [07](07-engineering-challenges.md) §7.5.

---

## 2d.7 Monaco editor (live coding)

`mockmind/frontend/components/CodeEditorPanel.tsx` → `CodeEditorPanel()` (L14+): a full-screen modal with the problem statement, the Monaco editor (`@monaco-editor/react`, `vs-dark`, minimap off, `automaticLayout`), and a **Submit Code** button disabled when empty. On submit the page serializes `{question_id, code}`, encodes it with `TextEncoder`, sends it on the `code_submission` topic with `reliable: true`, and hides the editor. Incoming challenges are parsed from `TextDecoder` output inside `try/catch` (malformed messages are ignored).
**Behaviours to know [Verified unless marked]:**
- The editor's text lives in `useState(challenge.starter_code)`; that initial value is read only on first mount, which is fine because the panel unmounts on submit (`setChallenge(null)`) and a later challenge mounts a fresh instance. Receiving a *second* `show_editor` while the panel is open would update the props but **not** the editor text (no `key` on the component) **[Inference from React semantics]**.
- `@monaco-editor/react` loads the Monaco runtime from a CDN by default: the installed `@monaco-editor/loader` config points at `https://cdn.jsdelivr.net/npm/monaco-editor@0.55.1/min/vs` (`node_modules/@monaco-editor/loader/lib/es/config/index.js`). So the editor needs internet access to jsDelivr from the candidate's browser, and an offline or CDN-blocked environment shows a loading editor that never appears. Nothing in the repo overrides the loader path.
- The `language` string from the agent (e.g. `python`) is passed straight to Monaco; an unknown language name just gets plain-text highlighting **[Standard knowledge]**.
- Submit is disabled only when the code is blank; there is no confirmation, no "run tests" button (tests are hidden and executed server-side), and no way to reopen the panel after submit.
- The panel is `fixed inset-0 z-40`, above the page but below the z-50 overlays; the voice orb remains running behind a dimmed backdrop, so the candidate can talk to Alex while coding.

---

## 2d.8 Tailwind CSS and the design language

Utility classes are applied directly (e.g. `rounded-xl`, `bg-white/5`, `hover:`, `sm:`, `print:hidden`). `tailwind.config.ts` scans `app/`, `components/`, `hooks/`, `lib/` and defines `font-display`/`font-sans`. `app/globals.css` adds keyframe animations (aurora drift, fade-in-up, shimmer, wave, blink, ping), a rotating conic-gradient border using `@property --angle`, a `prefers-reduced-motion` override, and print rules. Design guidance files (`DESIGN.md`, `CLAUDE.md`) are documentation for contributors/assistants, not runtime code. **[Verified]**
**Tailwind in 60 seconds:** instead of writing CSS rules you compose small single-purpose classes: `px-4` is horizontal padding of 1rem, `rounded-xl` a corner radius, `bg-white/5` white at 5 % opacity, `sm:grid-cols-2` applies only from the `sm` breakpoint up, `hover:scale-[1.03]` on hover, `print:hidden` only when printing. At build time Tailwind scans the files named in `tailwind.config.ts` `content` and emits CSS for only the classes it finds, which is why building class names dynamically (`'bg-' + color`) would silently fail; this repo avoids that by using full literal class strings (for example the `ORB_CONFIG` entries).
**Custom CSS that is *not* Tailwind [Verified, `app/globals.css`]:** keyframes `aurora-drift-1..3`, `fade-in-up`, `shimmer`, `float-y`, `spin-angle`, `wave`, `blink-dot`, `ping-soft`; the `.animate-fade-in-up` class; an `@property --angle` registered custom property that lets a conic-gradient border rotate (`.glow-border`); a `@media (prefers-reduced-motion: reduce)` block (L163); a `@media print` block (L174+). Fonts: `Space_Grotesk` (weights 500/600/700, `--font-display`) and `Inter` (`--font-sans`) via `next/font/google` in `layout.tsx`, which downloads and self-hosts the fonts at build time **[Standard knowledge]**.

---

## 2d.9 Report page: polling and "Save as PDF"

- **Polling [Verified]:** first fetch on mount; while `polling`, `setTimeout(fetchReport, 4000)`; stop on ready/failed or after 30 attempts (~2 min). `getReport()` maps HTTP **404→pending**, **502→failed**, **200→ready** (`lib/api.ts` L31–38). **Failed** shows *Try again*, which calls `retryReport()` (`POST /api/report/{id}/retry`) and restarts polling.
- **PDF:** a button calls `window.print()`. `@media print` rules in `globals.css` switch `.report-root` to dark-on-white, remove shadows/gradients (except `.print-keep` bars), and avoid page breaks inside cards; `print:hidden` hides controls.
- **Defensive rendering:** arrays use `?? []` so a missing field doesn't crash.
- **Four render states [Verified]:** (1) *generating*: spinner copy "This takes about 15-30 seconds" (a hard-coded estimate, **[Not measured]**); (2) *timed out*: after 30 polls shows "Report is taking longer than expected." and a **Try again** button that calls the same `handleRetry`; (3) *failed*: 502 from the API -> "We couldn't generate your report" with **Try again** and **New interview** (`/setup`); (4) *ready*: `ScoreBar` per category (colour thresholds: >= 7 green, >= 5 amber, else red; width = score/10), recommendation badge from `RECOMMENDATION_CONFIG` (`strong_yes`->"Strong Hire", `yes`->"Hire", `maybe`->"Maybe", `no`->"No Hire"), summary, strengths, improvements, per-question feedback. An unexpected `hiring_recommendation` value would make `RECOMMENDATION_CONFIG[...]` undefined; the backend's `validate_report` restricts it to those four.
- A network error while polling (`catch {}`) is swallowed and polling continues; the page cannot distinguish "backend down" from "still generating" until the 30-poll timeout.
- **Worked example:** interview ends at 12:00:00; the agent's pipeline needs ~25 s. Polls at 0, 4, 8 ... s return 404 -> `{status:'pending'}`. At ~25 s the report key exists -> 200 -> `setReport`, `setPolling(false)`, the chart renders. If the pipeline fails: 502 with `status=failed` -> failed view -> Try again -> `POST /retry` -> 202 -> `setPollCount(0); setPolling(true)` restarts the chain.

### Alternatives / trade-offs
A PDF library (headless Chrome, react-pdf) would give deterministic output but adds dependency/server work; browser print is zero-dependency but output depends on the user's browser. **[Inference]**

---

## 2d.10 Setup page and landing page behaviour

**`/` (`LandingPage`, server component):** static marketing content (`FEATURES` L14, `STEPS` L53, `STATS` L59, `WAVE_DELAYS` L65, `InterviewPreview` L67-130, `LandingPage` L131+), two `<Link href="/setup">` calls to action, and `AuroraBackground`. No data fetching, no hooks.
**`/setup` (`SetupPage`, client) [Verified]:**
- State: `resumeFile`, `jobDescription`, `useMemory`, `candidateEmail`, `interviewType` (`standard|coding`), `length`, `difficulty`, `focusWeakAreas`, `fileError`; the preparation state comes from `usePreparation()`.
- **Client-side validation mirrors the server's** (extension allowlist `.pdf .docx .doc .txt`, 5 MB, 15 000 chars; comment "Must match the limits enforced in backend/routers/prepare.py"). The values are duplicated in two languages and can drift. The client additionally requires the JD to be **more than 50 characters** (`canSubmit`), which the server does not.
- **Option gating:** *Length* is shown only for `standard` (the coding preset fixes 6 questions); `focusWeakAreas` is sent as `useMemory && focusWeakAreas`; the email is appended to the form only when `useMemory` is on and non-empty (`usePreparation.ts` L119-121). The email field is `type="email"` but not inside a `<form>`, so the browser does no format validation.
- **Progress UI:** each SSE event is appended to `events` and rendered with an icon from `STAGE_CONFIG` (unknown stage -> spinner icon); the bar width is `currentProgress`. The progress panel is hidden when `error` is set.
- **Ready state:** the `complete` event sets `sessionInfo`; the primary button flips from "Prepare My Interview" to "Start Interview", which `router.push('/interview/{sessionId}')`. Changing inputs after completion does not invalidate `sessionInfo`.
- **Quirk [Verified by reading]:** the error panel's **Start over** button clears the file and JD but the hook exposes no reset, so `error` stays set (and the panel stays visible) until the next `startPreparation()` call, which resets it.
- Cancel support exists in the hook (`cancel()` aborts the fetch) but `SetupPage` does not use it; there is no Cancel button.

## 2d.11 File inventory (frontend) [Verified]

| File | Role | Covered in |
|---|---|---|
| `app/layout.tsx` | root layout, fonts, metadata, global styles import | 2d.1, 2d.8 |
| `app/globals.css` | Tailwind layers, keyframes, glow/shimmer primitives, reduced-motion and print rules | 2d.8, 2d.9 |
| `app/page.tsx` | landing page (server component) | 2d.10 |
| `app/setup/page.tsx` | upload form, options, SSE progress | 2d.10 |
| `app/interview/[sessionId]/page.tsx` | mic gate, token, room UI, orb, overlays, coding modal | 2d.5-2d.7, 02c |
| `app/report/[sessionId]/page.tsx` | polling, report view, print | 2d.9 |
| `components/AuroraBackground.tsx` | decorative blurred gradient blobs (`subtle` prop lowers opacity); `aria-hidden`, `pointer-events-none`, uses the `aurora-drift-*` keyframes | 2d.8 |
| `components/CodeEditorPanel.tsx` | Monaco modal | 2d.7 |
| `hooks/usePreparation.ts` | SSE-over-POST client | 2d.4 |
| `hooks/useInterviewSession.ts` | **unused** state-mapping hook | 2d.2, 02c §2c.11 |
| `lib/api.ts` | `getSessionToken`, `getReport`, `retryReport`; `BACKEND_URL` from `NEXT_PUBLIC_BACKEND_URL` (default `http://localhost:8000`) | 2d.3, 2d.9 |
| `types/interview.ts` | hand-written API types | 2d.3 |
| `package.json`, `package-lock.json` | dependencies and scripts (`dev/build/start/lint`) | header |
| `next.config.js`, `tsconfig.json`, `tailwind.config.ts`, `postcss.config.js` | build/type/CSS config | 2d.1, 2d.3, 2d.8 |
| `Dockerfile`, `.env.local.example` | image build; the two `NEXT_PUBLIC_*` variables (values not recorded here) | 2d.1 |
| `next-env.d.ts`, `tsconfig.tsbuildinfo` | generated by Next/TypeScript; not hand-written (the latter is not tracked by git) | excluded |
| `public/` | empty directory (see 2d.1 note) | 2d.1 |
| `node_modules/`, `.next/` | installed/generated | excluded |

## 2d.12 Interview connection
- Explain the SSE-over-POST parser and the chunk-boundary bug it avoids.
- Explain why the mic is requested before joining and how the 25 s timer works.
- Explain server vs client components with the actual pages.
- Explain what `RoomAudioRenderer` does and what happens without it.
- Explain the Reconnect limitation precisely (it restarts the interview).
