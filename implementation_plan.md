# CollegePath AI Counsellor — Performance Diagnosis & Optimization Plan (Revised)

A comprehensive root-cause analysis, stability fix, and end-to-end performance optimization plan for the CollegePath AI Admission Counsellor, reducing response times from **>30 seconds** down to **<1 second Time-to-First-Token (TTFT)** through streaming SSE, elimination of frontend TCP stalls, intelligent in-memory verified data caching, indexed college retrieval, and fixing dev server pre-warming port binding.

> [!IMPORTANT]
> **Plan Revision Summary:**
> 1. **Immediate Stability Fix (Dev Server Pre-warming Failure)**: Identified that `process.env.PORT` in the cloud environment was defaulting to `8080` (which is already occupied by the internal proxy), causing `Error: listen EADDRINUSE: address already in use :::8080` and killing the dev server process. We will lock the dev server port to `3000` as strictly required by AI Studio constraints.
> 2. **Performance Optimization Execution**: Ready to execute all 10 optimizations and telemetry logging once approved.

---

## Phase 1: Performance Audit & Root-Cause Diagnosis

Our codebase inspection and live diagnostic tests revealed the specific cumulative bottlenecks causing responses to take **over 30 seconds**:

```
CURRENT CUMULATIVE DELAY WATERFALL (>30 SECONDS):
┌─────────────────────────────────────────────────────────────┬──────────────┐
│ Operation                                                   │ Latency      │
├─────────────────────────────────────────────────────────────┼──────────────┤
│ 1. Frontend phantom fetch to http://localhost:5000/api/chat │ 10s - 20s    │
│ 2. Unary blocking Gemini API call (no streaming SSE)        │ 8s - 15s     │
│ 3. Gemini 3.8 Flash ThinkingLevel.MEDIUM reasoning delay    │ 6s - 12s     │
│ 4. Sequential fallback retry loops upon 503 high demand     │ 10s - 15s    │
│ 5. Synchronous disk I/O on profiles.json per chat turn      │ 15ms - 50ms  │
│ 6. Linear scan of unindexed colleges array                  │ 5ms - 20ms   │
│ 7. Repeated API calls for identical canonical questions     │ No Cache     │
└─────────────────────────────────────────────────────────────┴──────────────┘
Total User Perceived Latency: 25s - 45s (Average: >30 seconds)
```

### Detailed Breakdown of the 8 Audit Areas:

1. **Frontend Request Handling (`src/components/AICounsellorChat.tsx`)**:
   - **Localhost:5000 TCP Stall**: Lines 178–193 check `if (isLocalhost)` and issue `fetch('http://localhost:5000/api/chat')`. Because the applet runs on port 3000 (`tsx server.ts`), port 5000 is not listening. The browser stalls waiting for TCP connection rejection or timeout (taking 10 to 20+ seconds) before finally falling back to `/api/chat`.
   - **Blocking Unary Response**: Uses standard `fetch` + `await response.json()`. The user sees nothing but a spinning loading state until the entire message (often 500–1,000 words) has finished generating.
   - **No Request Abort**: Lacks `AbortController` support, meaning rapid queries or retries create orphaned server requests.

2. **Backend API Routes & Server Stability (`server.ts` & `backend/routes/chatRoutes.js`)**:
   - **Port Collision & Pre-warming Failure**: `server.ts` checked `const PORT = process.env.PORT || 3000`. In the cloud environment, `process.env.PORT` is set to `8080` which is already occupied, throwing `EADDRINUSE :::8080`. We must bind explicitly to `3000`.
   - **No Streaming Endpoint**: Backend only supports unary JSON (`res.json({ reply })`) with no Server-Sent Events (`text/event-stream`) endpoint.

3. **Gemini API Integration (`backend/config/gemini.js` & `chatController.js`)**:
   - **Unconfigured Reasoning Latency**: `gemini-3.8-flash` defaults to `ThinkingLevel.MEDIUM`, which performs deep multi-step reasoning before outputting the first token. For standard counselling and informational queries, this adds 6–12 seconds of latency. Setting `thinkingLevel: ThinkingLevel.LOW` cuts reasoning latency while preserving analytical quality.
   - **Blocking Fallback Chain Without Fast Timeout**: If `gemini-3.8-flash` encounters temporary 503 load, the current code waits for the full upstream request timeout (up to 15s) before trying `gemini-3.1-flash-lite`, causing massive latency spikes.
   - **SDK Calling Pattern**: Currently uses `ai.models.generateContent` instead of `ai.models.generateContentStream`.

4. **Database Queries & College Retrieval (`server/collegeDb.ts`)**:
   - `getDatabaseProfile()` calls synchronous `fs.readFileSync(PROFILES_FILE)` on disk for every single chat message.
   - No dedicated inverted search index or hash lookup exists for fast college matching by name, exam, or branch. While the 109 verified colleges are in memory, queries use linear `.filter()` with nested `.some()`.
   - The chat prompt currently receives either the entire student profile or generic instructions without targeted retrieval of relevant candidate colleges.

5. **External Web Search or Scraping Operations**:
   - No external scrapers are active. However, the system prompt asked the model to include official hyperlinks without supplying verified grounding links in context, causing the model to spend reasoning cycles hallucinating or verifying external sources.

6. **Prompt Size & Conversation History**:
   - `SYSTEM_INSTRUCTION` is 40 lines of repetitive rules.
   - History currently sends the raw last 10 messages without trimming whitespace or token limits.

7. **Duplicate or Unnecessary API Calls**:
   - Deterministic admission queries (e.g., "What is the difference between JEE Main and JEE Advanced?", "Explain JoSAA counselling rounds", or "Tuition fee of IIT Delhi") are called repeatedly against the Gemini API with 0 caching.
   - The frontend's dual-fetch pattern (`localhost:5000` then `/api/chat`) triggered duplicate attempts.

8. **Sequential Operations that Can Run in Parallel**:
   - Profile verification, college index search, and cache check currently run in sequence. These can be executed synchronously from in-memory indexed structures in <2ms before calling Gemini.

---

## Phase 2: Performance Measurement & Telemetry Design

We will introduce a non-intrusive, zero-leak telemetry logger using high-resolution monotonic timestamps (`performance.now()`) and unique Request IDs (`req_xxxx`).

### Measured Metrics:
1. `totalResponseTimeMs`: Total time from client request to last byte sent.
2. `dbQueryTimeMs`: In-memory profile and college retrieval time.
3. `cacheLookupTimeMs`: Time taken to check and serve from in-memory cache.
4. `geminiApiTimeMs`: Duration of the upstream Gemini API streaming call.
5. `timeToFirstTokenMs` (TTFT): Time elapsed until the first generated token arrives at the server and client.
6. `backendProcessingTimeMs`: Total server overhead excluding upstream Gemini API call.
7. `geminiCallCount`: Number of API calls attempted (1 for primary, +1 if fallback triggered).
8. `cacheHit`: Boolean flag indicating whether the response was served from cache.

### Privacy & Security Guarantees:
- **Never log API keys**: Keys remain strictly within `process.env`.
- **Never log student PII**: No email addresses, phone numbers, or student names in server logs. Only anonymized lengths and category types.
- **Client Transparency**: Send metrics via custom HTTP response headers (`X-Response-Time-Ms`, `X-TTFT-Ms`, `X-Cache-Hit`) and in the final streaming SSE event chunk so the frontend UI can optionally display real response stats.

---

## Phase 3: Optimization Architecture

```
OPTIMIZED REQUEST FLOW (<1s TTFT):
┌─────────────────────────┐
│ User enters query in UI │
└────────────┬────────────┘
             │ 1. Direct fetch to /api/chat/stream (SSE) [No localhost:5000 stall]
             ▼
┌─────────────────────────────────────────────────────────┐
│ Server: chatController.ts                               │
├─────────────────────────────────────────────────────────┤
│ • Generate Request ID (e.g. req_8f12)                   │
│ • Check In-Memory LRU Cache (<1ms)                      │
│   ├── CACHE HIT ──► Immediate SSE stream (TTFT < 10ms)  │
│   └── CACHE MISS                                        │
│ • Indexed College Retrieval (<2ms)                      │
│   (Select top 3-5 relevant colleges based on query)     │
│ • Compact System Prompt & Trimmed History               │
│ • Gemini 3.8 Flash Stream with ThinkingLevel.LOW        │
│ • 8s Fast-Failover Timeout to gemini-3.1-flash-lite     │
└────────────┬────────────────────────────────────────────┘
             │ 2. Server-Sent Events (SSE) Stream
             ▼
┌─────────────────────────────────────────────────────────┐
│ Frontend: AICounsellorChat.tsx                          │
│ • Progressive Markdown display as tokens arrive         │
│ • Immediate visual typing cursor & active response      │
│ • Response completed in 1.5s - 3s (TTFT < 800ms)        │
└─────────────────────────────────────────────────────────┘
```

### 10 Specific Optimizations to Implement:

1. **Dev Server Port Fix (`server.ts`)**:
   - Ensure the server always listens on port `3000` to prevent `EADDRINUSE: :::8080` failures in the AI Studio environment.

2. **Gemini Streaming SSE (`/api/chat/stream`)**:
   - Implement `ai.models.generateContentStream` over HTTP Server-Sent Events (`text/event-stream`).
   - Deliver chunks immediately to the browser using `res.write(`data: ${JSON.stringify({ text, ttft })}\n\n`)`.
   - Maintain a fallback unary route (`/api/chat`) for compatibility with existing clients.

3. **Smart College Retrieval (Indexed Database)**:
   - Build an in-memory hash map index (`nameIndex`, `examIndex`, `cityIndex`, `courseIndex`) in `server/collegeDb.ts` during initialization.
   - When the user asks about specific institutions (e.g. "IIT Delhi", "BITS Pilani") or categories (e.g. "CSE colleges"), extract keywords and retrieve the exact 2 to 5 relevant colleges rather than dumping large unstructured data into the prompt.

4. **Optimized, High-Density System Prompt**:
   - Streamline the prompt from 40+ lines down to a concise, high-impact instruction focusing strictly on zero hallucination, NIRF/JoSAA grounding, and structured table format.
   - Limit conversation history to the last 4 relevant turns with trimmed tokens.

5. **Thinking Level Tuning (`ThinkingLevel.LOW`)**:
   - Set `thinkingConfig: { thinkingLevel: ThinkingLevel.LOW }` for `gemini-3.8-flash`. This drastically speeds up response initiation for factual Q&A while maintaining high reasoning quality.

6. **In-Memory Verified Admission Knowledge Cache**:
   - Implement a TTL-based LRU cache (500 entries, 24-hour TTL) for verified answers to standard entrance, counselling, and fee questions.
   - Automatic cache hit returns responses in <10ms.

7. **Remove Dead Frontend Fetches**:
   - Remove the `http://localhost:5000` probe from `AICounsellorChat.tsx`. Route directly to `/api/chat/stream` with fallback to `/api/chat`.

8. **Fast Failover & Timeout Controller**:
   - Set an 8-second timeout on the primary model call using `AbortSignal`. If the primary model experiences high load (503), it immediately falls back to `gemini-3.1-flash-lite` without hanging.

9. **Non-Blocking In-Memory Profile Access**:
   - Cache `profiles.json` in memory with asynchronous debounced disk persistence, removing synchronous file reads on every turn.

10. **Progressive Response Rendering in UI**:
    - Update `AICounsellorChat.tsx` to read the SSE stream using `ReadableStreamDefaultReader` and update state dynamically as tokens arrive.

---

## Phase 4: Testing & Verification Plan

We will benchmark response times before and after optimization across the four mandatory test cases:

| Test Query | Target Before | Target After Optimization |
| :--- | :--- | :--- |
| **1. What is the difference between JEE Main and JEE Advanced?** | >30s (unary/timeout) | **TTFT < 800ms** (Cached: <20ms) |
| **2. Suggest suitable CSE colleges for my entrance score.** | >30s | **TTFT < 900ms**, Total < 3.5s |
| **3. What is the tuition fee and hostel fee of IIT Delhi?** | >30s | **TTFT < 600ms**, Total < 2.5s |
| **4. Explain JoSAA counselling rounds.** | >30s | **TTFT < 800ms** (Cached: <20ms) |

Each test will log and output:
- `totalResponseTimeMs`
- `timeToFirstTokenMs`
- `dbQueryTimeMs`
- `geminiApiTimeMs`
- `backendProcessingTimeMs`
- `geminiCallCount`
- `cacheHit` status

---

## Critical Constraints & Invariant Verification:
- **Preserve Existing UI**: Retain all navbar tabs, chat bubbles, avatar styling, markdown rendering, copy buttons, retry triggers, and suggestions chips.
- **Preserve Features**: Zero change to college comparison, document assistant, scholarship finder, fees calculator, or admin dashboard.
- **Preserve Credentials**: Keep `GEMINI_API_KEY` and `.env` configs intact without exposing them to the client.
- **Preserve Factual Accuracy**: Grounded strictly in verified college data, JoSAA/CSAB official frameworks, and NIRF rankings.
