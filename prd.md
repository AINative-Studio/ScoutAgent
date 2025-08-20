# PRD — ZeroMemory Demo App (“Scout”)

## 1. Summary

A minimal agent demo that **saves**, **replays**, and **uses** user memory to produce contextual responses. Built to teach AINative memory concepts and provide a repeatable classroom lab.

## 2. Objectives & Success Metrics

* Show **vector+recency hybrid replay** affecting responses (✅ visual proof).
* Students can run the app locally with their own keys in <10 minutes (✅ quickstart).
* Replay panel shows **which** memories influenced the answer (✅ transparency).

## 3. Users & Scenarios

* **Agent builders / AI UX engineers** evaluating agent memory.
* Scenario A: Preference carry-over (vegan, no early meetings).
* Scenario B: Task/fact recall (pitch date, teammate names).
* Scenario C: Scope switch (session vs global) to illustrate memory boundaries.

## 4. Non-Goals

* Full CRM-grade profile, embeddings tuning UI, or advanced extractors.
* Multi-agent orchestration; this demo focuses on **single agent memory**.

## 5. Functional Requirements

**F1. Chat with Memory Injection**

* `/chat` accepts `user_id, session_id, agent_id, message, use_memory, top_k`.
* If `use_memory=true`, backend replays memory and injects into prompt.

**F2. Save Memory Entries**

* On each turn, a basic extractor marks lines as `preference`, `task`, or `fact`.
* Memory saved with `metadata` (source, tags, timestamps, origin turn).

**F3. Replay (Vector + Recency)**

* Overfetch via SDK semantic search; re-rank with recency half-life (default 72h).
* Filters: `type_in` (`preference|fact|task|mood`), `scope` (current session/global).

**F4. Feedback**

* `/feedback` accepts `{memory_id, signal: +1|-1, reason?}`; adjust `priority` or store a signal record.

**F5. Memory Panel**

* Show list of replayed memories with `type, truncated text, score`.

**F6. Controls**

* Toggle **Use memory** ON/OFF.
* Switch **Scope**: Session / Global.
* Slider **Top-k**: 1–8.

**F7. Health & Debug**

* `/memories` (dev only) returns last N saved entries (masked content if `SAFE_MODE=true`).
* `/healthz` returns 200 with adapter type (sdk|mock).

**F8. MCP Injection (Optional)**

* Provide a sample payload that forwards replayed memory as contextual hints to a tool.

## 6. Non-Functional Requirements

* **Performance**: P50 endpoint latency ≤ 800ms (excluding LLM call).
* **Compatibility**: Python 3.10+, Node 18+.
* **Security**: API key in env var; CORS limited to localhost by default.
* **Privacy**: Toggle to redact memory text in logs; default is redact.

## 7. Data Model (Memory)

```json
{
  "id": "string",
  "user_id": "string",
  "agent_id": "string",
  "session_id": "string",
  "type": "preference|fact|task|mood|note",
  "content": "string",
  "metadata": {
    "source": "user|agent|system",
    "tags": ["string"],
    "origin_turn_id": "string",
    "ts": "ISO-8601"
  },
  "priority": 0
}
```

## 8. API Contracts (Demo Backend)

### POST `/chat`

**Request**

```json
{
  "user_id":"u1",
  "session_id":"s1",
  "agent_id":"scout",
  "message":"Plan lunch + review",
  "use_memory": true,
  "top_k": 5
}
```

**Response**

```json
{
  "reply": "Let's pick a vegan-friendly spot. Your pitch is Friday 10am...",
  "replayed_memory": [
    {"id":"m1","type":"preference","content":"User is vegan","score":0.87},
    {"id":"m2","type":"fact","content":"Pitch on Friday 10am","score":0.82}
  ]
}
```

### POST `/feedback`

**Request**

```json
{"memory_id":"m2","signal":"+1","reason":"Relevant to planning"}
```

**Response**

```json
{"ok": true}
```

### GET `/memories?scope=session|global&limit=50`  (dev)

**Response**

```json
{"items":[{"id":"m1","type":"preference","content":"..."}]}
```

## 9. Prompt Template

**System**

```
You are Scout, a helpful, empathetic agent.
Use MEMORY only when relevant. If MEMORY conflicts with new instructions, ask what to follow.
```

**Injected Memory Block**

```
[MEMORY CONTEXT]
- preference: User is vegan.
- fact: Pitch on Friday 10am.
[/MEMORY CONTEXT]
```

**User**: raw message

## 10. UI/UX Spec

* **Layout**: Two-pane. Left = chat; Right = “Replayed Memory”.
* **Controls**: Top bar with Use-memory toggle, Scope switch (Session/Global), Top-k slider, and a “Clear session” button.
* **States**:

  * Empty memory → panel says “No relevant memory yet.”
  * Loading → skeleton rows.
  * Error → toast with retry.
* **Accessibility**: Keyboard nav for send, sliders, and toggles; ARIA labels on controls.

## 11. Testing

* Unit: adapter functions (SDK mocked).
* Integration: `/chat` returns replayed memory; toggle OFF changes response content.
* Snapshot: prompt builder includes memory block when enabled.
* Smoke: seed script + three demo turns produce expected panel items.

## 12. Deployment & Run

* **Backend**: `uvicorn backend.app:app --reload`
* **Frontend**: `npm i && npm run dev`
* **Env**: copy `.env.example` → `.env`, set `AINATIVE_API_KEY` and `LLM_URL`.

## 13. Acceptance Criteria

* With memory ON, demo scenarios produce replies referencing the correct memories, and the panel lists those memories with scores.
* With memory OFF, replies become generic and do not reference stored facts/preferences.
* Feedback endpoint returns `ok=true` and records the signal.

## 14. Risks / Open Items

* If SDK method names differ, update the **adapter** only; backend/UI remain unchanged.
* If replay volume is high, cap overfetch and paginate panel results.

---

