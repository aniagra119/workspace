# Agent Context — Manas Ecosystem

> This file is for AI agents (Gemini, Claude, GPT, etc.) working on this codebase.
> Read this first before making any changes. It summarises every critical design decision so you don't have to infer from code alone.

---

## What This Project Is

**Manas** is a conversational AI Operating System. It is NOT a chatbot wrapper. It is a stateful, event-driven backend that:
1. Receives messages from a **transport layer** (Discord, Slack, Telegram, CLI)
2. Routes intent through a **5-Node LangGraph ReAct graph**
3. Reads/writes structured data to a **storage layer** (Google Sheets, Postgres, SQLite)
4. Returns a response back via the transport layer

The LangGraph graph knows nothing about Discord or Google Sheets. It only talks to Protocol interfaces.

---

## Non-Negotiable Constraints

1. **Append-only.** Never add `update_row` or `delete_row` to the `DatabaseClient` Protocol for user persona tabs. Historical data cannot be modified by the AI.
2. **No hardcoded IDs.** No spreadsheet IDs, Discord channel IDs, or API keys anywhere in application code. All config lives in `.env` → `core/config.py`.
3. **Protocol before implementation.** If you add a new external dependency, define the Protocol interface in `libs/adapters-db/protocols.py` or `libs/adapters-transport/protocols.py` before writing the implementation.
4. **No global state.** `AgentState` is the only mutable state during a graph run. Never use module-level globals or class-level caches inside LangGraph nodes.
5. **Redis idempotency.** Every inbound message must be checked against `manas:seen:{message_id}` before enqueuing. This prevents double-logging from retried webhooks.

---

## Repository Structure

```
manas/                          github.com/aniagra119/manas
├── apps/manas-server/          The AI OS application
│   ├── api/                    Inbound webhook handlers (signature verification first)
│   ├── agents/
│   │   ├── state.py            AgentState TypedDict — the only mutable graph state
│   │   ├── graph.py            StateGraph wiring (5 nodes)
│   │   ├── bootstrap/          Parallel schema + config readers (Redis TTL cached)
│   │   └── cognitive/          SemanticGatewayNode, ExecutionAgentNode, ResponseSynthesizerNode
│   ├── manthan/                Nightly APScheduler engine (never touches live graph)
│   └── workers/                Persistent Redis queue consumers
└── libs/
    ├── adapters-db/            DatabaseClient Protocol + Sheets/SQLite/Postgres adapters
    └── adapters-transport/     TransportAdapter + TransportEnvelope + Discord/Slack adapters
```

---

## The 5-Node Graph

```
Bootstrap (parallel, Redis cached):
  SchemaInspectorNode  →  reads _MANAS_SCHEMA + tab headers
  ConfigManagerNode    →  reads _MANAS_CONFIG + rules

Cognitive:
  SemanticGatewayNode  →  generates tool_calls[] from user input + dynamic Pydantic schemas
  ExecutionAgentNode   →  ReAct loop iterating over tool_calls (read_db, write_db, fetch_history)
  ResponseSynthesizerNode  →  reads trajectory + tool_outputs, writes synthesized_response
```

Fast path: Gateway → Synthesizer (write goes to Redis write_queue, no ExecutionAgent)
Full path: Gateway → ExecutionAgent → Synthesizer

---

## Key Data Contracts (Google Sheets Control Plane)

| Tab | Purpose |
|:---|:---|
| `_MANAS_CONFIG` | Persona-to-channel mapping + intercept rules |
| `_MANAS_SCHEMA` | Column capability registry (type, owner, extractable flag) |
| `_MANAS_TRACKERS` | Tracker lifecycle (proposed → active → deprecated) |
| `_MANAS_VIEWS` | Computed view registry (formula, materialized, SQL view) |
| `_MANAS_META` | Operational telemetry (write-only from graph, read by Manthan) |

User persona tabs (e.g. `finance`, `health`) are **flat append-only logs**. The AI can only `append_row`.

---

## Redis Key Namespace

```
manas:seen:{message_id}              Idempotency — 24hr TTL
manas:{channel_ref}:in_flight        Job lock — 120s TTL
manas:{channel_ref}:msg_buffer       Coalesced messages while in_flight
manas:{channel_ref}:bootstrap_ready  Cached bootstrap data — configurable TTL
manas:schema_lock                    Distributed lock for structural Sheets writes
manas:job_queue                      Main FIFO job queue (LPUSH/RPOP)
manas:write_queue                    Write debounce queue → Sheets batchUpdate
```

---

## Current Development Phase

**Phase 0 — Registry & Onboarding** is the current target.

Files to write next:
- `libs/adapters-db/protocols.py` — DatabaseClient, VectorClient Protocol stubs
- `libs/adapters-db/sqlite.py` — SQLiteClient for registry.db only
- `apps/manas-server/core/registry.py` — lookup/save user→spreadsheet_id in SQLite
- `apps/manas-server/core/redis.py` — async Redis connection pool
- `apps/manas-server/api/webhook.py` — verify → registry lookup → enqueue → return type:5

See `CODING_PLAN.md` for full phase breakdown and branch strategy.

---

## What Phase 1 Looks Like When Done

```bash
docker compose up -d
# Send a fake Discord webhook to localhost:9000
# See the job appear in Redis queue
# See the bot respond "Logged ₹850 (Dinner)" in Discord
# See a new row appear in the user's Google Sheet
```

---

## Versioning

SemVer 2.0 across all repos.
- MAJOR = breaking Protocol change (affects multiple repos)
- MINOR = new node, new adapter, new capability
- PATCH = bug fix, config change, doc update

Current version: `v0.1.0-pre-restructure` (tagged)
Next milestone: `v0.2.0` — first fully runnable Phase 0+1
