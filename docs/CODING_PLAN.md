# Manas: Coding Execution Plan

> This document defines exactly what to code, in what order, in which branch/repo, and when a piece of code graduates to a standalone library.

---

## Repository Roles (Coding Context)

| Repo | What you code here |
|:---|:---|
| `manas` | The AI OS application — FastAPI, LangGraph graph, workers, Manthan Engine |
| `developer-palette` | Branch templates only. No application code. |
| `manas-storage` *(future)* | Extracted once the Sheets/SQLite client is stable and used by 2+ projects |

---

## What Makes a Library vs Application Code?

**Extract to a standalone library when:**
- The code is useful with zero knowledge of Discord, LangGraph, or Manas-specific logic
- You find yourself copy-pasting it into a second project
- It has a clean, stable Protocol boundary (i.e. it is already a Python `Protocol` class)
- It could reasonably be `pip install`-ed by a stranger and used independently

**Keep in the application when:**
- The code references `AgentState`, LangGraph nodes, or Discord-specific types
- It is business logic specific to how Manas works (e.g. `SemanticGatewayNode`)
- It is early/unstable and changing rapidly

**Decision for now:**
- `libs/manas-storage/` → stays inside the `manas` repo for Phase 1. Will be extracted to its own GitHub repo when `developer-palette` also needs to import it.
- `libs/manas-transport/` → same. Extract when a second transport (Slack/Telegram) is added.

---

## Branch Strategy (inside `manas` repo)

```
main                    ← Stable, always deployable. Tagged with SemVer.
│
├── dev                 ← Active development branch. All PRs merge here first.
│   ├── feat/phase-0-registry       ← SQLite registry + Drive provisioning
│   ├── feat/phase-1-protocols      ← manas-storage and manas-transport libs
│   ├── feat/phase-1-graph          ← 5-node LangGraph wiring
│   ├── feat/phase-2-webhook        ← FastAPI router + Ed25519 + Redis queue
│   └── feat/phase-3-manthan        ← Manthan Engine + APScheduler
│
└── [tags]
    ├── v0.1.0-pre-restructure  ← Backup before docs overhaul (exists now)
    └── v0.2.0                  ← First runnable Phase 0+1 milestone
```

**Rule:** Never commit directly to `main`. Always branch from `dev`, code, then merge back to `dev`. Only merge `dev → main` when a full phase is complete and tested.

---

## Phased Coding Execution

### Phase 0 — Registry & Onboarding (Start Here)
**Branch:** `feat/phase-0-registry`  
**Goal:** Bot can receive a Discord message, look up the user in SQLite, and either route them to onboarding or attach their `spreadsheet_id` to the job.

Files to write:
```
libs/manas-storage/
├── protocols.py          # DatabaseClient, VectorClient Protocol stubs
├── sqlite.py             # SQLiteClient — registry.db operations only
└── pyproject.toml

apps/manas-server/
├── core/
│   ├── config.py         # Pydantic Settings (already exists, needs cleanup)
│   ├── registry.py       # SQLite query/write for user → spreadsheet_id mapping
│   └── redis.py          # Async Redis connection pool
└── api/
    └── webhook.py        # Minimal Discord webhook handler (verify → lookup → queue)
```

Done when: Running the server locally, sending a fake Discord webhook, seeing the job land in Redis.

---

### Phase 1 — Protocols & Bootstrap Nodes
**Branch:** `feat/phase-1-protocols`  
**Goal:** The full `DatabaseClient` and `TransportAdapter` are implemented. Bootstrap nodes can read `_MANAS_SCHEMA` and `_MANAS_CONFIG` from a real Google Sheet.

Files to write:
```
libs/manas-storage/
└── sheets.py             # GoogleSheetsClient implementing DatabaseClient

libs/manas-transport/
├── protocols.py          # TransportAdapter, ChatHistoryClient stubs
├── envelope.py           # TransportEnvelope dataclass
├── discord.py            # DiscordAdapter implementation
└── pyproject.toml

apps/manas-server/
└── agents/
    ├── state.py          # AgentState TypedDict
    ├── bootstrap/
    │   ├── schema_inspector.py
    │   └── config_manager.py
    └── graph.py          # StateGraph skeleton (bootstrap nodes only, no cognitive yet)
```

Done when: Bootstrap nodes read a real Google Sheet and populate `AgentState` correctly. Verified via a local test script.

---

### Phase 2 — The 5-Node ReAct Graph
**Branch:** `feat/phase-1-graph` (merge into `dev` when done)  
**Goal:** Full LangGraph graph runs end-to-end. Send a Discord message, get a response back, see a row appear in Google Sheets.

Files to write:
```
apps/manas-server/
└── agents/
    └── cognitive/
        ├── semantic_gateway.py    # DynamicRouteIntent + pydantic.create_model
        ├── execution_agent.py     # ReAct tool loop
        └── response_synthesizer.py

apps/manas-server/
└── workers/
    ├── agent_worker.py            # Redis consumer → runs graph
    └── write_debouncer.py         # Redis write_queue → Sheets batchUpdate
```

Done when: `docker compose up` → send Discord message → bot replies → row in Sheets.  
**This is the v0.2.0 milestone.**

---

### Phase 3 — Manthan Engine
**Branch:** `feat/phase-3-manthan`  
**Goal:** Nightly cron runs, reads telemetry, proposes a tracker via Discord.

```
apps/manas-server/
└── manthan/
    ├── engine.py              # APScheduler entrypoint
    └── telemetry_analyzer.py  # Pattern detection
```

---

### Phase 4 — Library Extraction (when ready)
When `developer-palette` needs to test against `manas-storage`:
1. Move `libs/manas-storage/` out of this repo
2. Create `github.com/aniagra119/manas-storage` as a standalone public repo
3. Replace the `uv` local path reference with a Git URL reference
4. Update `developer-palette/pyproject.toml` to depend on it

---

## Coding Rules

1. **Test before merge.** Each `feat/` branch must have at least one working local test before merging to `dev`.
2. **Protocol first.** Write the `Protocol` stub before the implementation. This forces clean interface design.
3. **No hardcoded IDs.** All config goes through `core/config.py` Pydantic Settings and `.env`.
4. **Append-only for user data.** The `DatabaseClient` must never expose an `update_row` or `delete_row` method on persona tabs.
5. **One `pyproject.toml` per library.** Each `libs/` package is independently installable.
