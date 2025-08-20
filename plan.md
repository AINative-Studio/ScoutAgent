# Implementation Plan — ZeroMemory Demo Agent (Scout)

## Goals

* Showcase AINative **Memory for Agents** via Python SDK in a tiny, live-codable app.
* Demonstrate **save → replay (vector+recency) → prompt injection → response**.
* Provide obvious “with vs without memory” UX and a scoreboard of which memories were used.

## Constraints & Assumptions

* Use **AINative Python SDK** (wrap in a thin adapter to decouple from SDK surface changes).
* Keep infra minimal: **FastAPI** backend + **Vite/React** SPA.
* LLM can be local (Ollama) or a hosted endpoint; app is LLM-agnostic.
* Works offline with a **mock SDK** if the real API key isn’t available.

## High-Level Flow

1. User chats → backend calls **MemoryReplay** (semantic search + recency decay) via SDK.
2. Backend injects replayed memory into the LLM prompt → gets reply.
3. Backend saves new memory candidates (simple extractor) via SDK.
4. UI shows replayed memory (type, text, score) and a “Use memory” toggle.

## Architecture

```
frontend (Vite/React)
  ├─ Chat view
  └─ Memory panel (read-only)

backend (FastAPI)
  ├─ /chat        -> adapter.replay() -> LLM -> adapter.add()
  ├─ /feedback    -> adapter.feedback()
  ├─ /memories    -> adapter.list() (for debugging/demo)
  └─ /healthz

AINativeAdapter (Python)
  ├─ add_memory(user_id, agent_id, session_id, type, text, tags, metadata)
  ├─ replay_memory(user_id, agent_id, session_id, query, top_k, filters, strategy)
  ├─ list_memories(tags | session scope)
  └─ feedback(memory_id, signal, reason)
```

## Scoring (Replay Strategy)

* SDK semantic search (vector similarity) → **overfetch** K×3.
* Re-rank with **recency decay** (half-life default: 72h).
* Final score = `similarity * decay_weight`; take **top-k**.

## Repo Structure

```
scout/
  backend/
    app.py
    adapter/
      ainative_adapter.py   # wraps the SDK
      mock_adapter.py       # fallback for demos without API key
    tests/
      test_adapter.py
      test_chat_e2e.py
    .env.example
  frontend/
    src/App.tsx
    src/components/Chat.tsx
    src/components/MemoryPanel.tsx
    src/lib/api.ts
    vite.config.ts
  README.md
```

## Environment & Config

* `AINATIVE_API_KEY`, `AINATIVE_PROJECT` (if needed), `LLM_URL`, `PORT`.
* Feature flags: `USE_MOCK_ADAPTER`, `MEMORY_HALFLIFE_HOURS`, `DEFAULT_TOP_K`.

## Work Breakdown (with Est. Effort)

1. **Adapter layer** (SDK + mock) — 0.5d

   * Implement `add_memory`, `replay_memory`, `list_memories`, `feedback`.
2. **Replay logic** — 0.25d

   * Overfetch, recency half-life, scoring, filters (type, session/global).
3. **FastAPI endpoints** — 0.5d

   * `/chat`, `/feedback`, `/memories`, `/healthz`; error handling.
4. **LLM client** — 0.25d

   * Minimal HTTP client; prompt builder with memory block.
5. **Frontend** — 0.75d

   * Chat UI + right-side memory panel; memory toggle; top-k slider; scope switch.
6. **Seed & scripts** — 0.25d

   * `seed_memories.py`, cURL examples, demo data.
7. **Testing** — 0.5d

   * Unit tests (adapter), integration test (`/chat` with mock), smoke runbook.
8. **Docs & Runbook** — 0.25d

   * README quickstart, live-demo script, troubleshooting.

*Total: \~3.25 dev-days (one strong dev can compress to a single focused day).*

## Quality Gates

* **P50 /chat** ≤ 800ms (mock LLM excluded); **P95** ≤ 1500ms.
* **Top-k replay correctness**: manual checks show expected memories appear with scores.
* **Toggle validation**: “Use memory” OFF produces materially different, generic replies.

## Telemetry & Logging

* Log replay request/response (IDs, scores, counts), not raw content by default.
* Count memory saves per session; track feedback (+1/−1) rates.
* Frontend: record toggle state and top-k changes (console is fine for demo).

## Live Demo Runbook (5–7 min)

1. Cold ask (memory off): “Lunch ideas + time to review pitch?” → generic.
2. Teach memory: “I’m vegan and hate early meetings; pitch is Friday 10am.”
3. Ask again (memory on): replay shows vegan + Friday; reply is contextual.
4. Show feedback (+1) on the most helpful memory.
5. Switch scope to **Global**; demonstrate cross-session recall.

## Risks & Mitigations

* **SDK surface mismatch** → **Adapter** isolates changes.
* **No API key / flaky network** → **Mock adapter** + seeded memories.
* **LLM variance** → Fix temperature; keep prompts deterministic.

---

