# Coding Standards & Practices

> Personal reference — for myself and for AI agents working on this codebase.
> Last updated: 2026-05-20

---

## Language & Runtime

- **Python 3.12+** — use `match` statements over long `if/elif` chains where intent is cleaner
- **`uv`** for all dependency management — never `pip install` directly
- **Type hints everywhere** — no untyped function signatures, ever
- **Pydantic for all data models** — never use plain `dict` as a function boundary

---

## Architecture Principles

### 1. Protocol First
Write the `Protocol` stub before the implementation. Every external dependency (database, transport, cache) must be hidden behind a Python `Protocol` class. This is non-negotiable.

```python
# ✅ Correct — define the boundary first
class DatabaseClient(Protocol):
    async def append_row(self, table_ref: str, row: dict) -> None: ...

# ❌ Wrong — coding directly against the implementation
from src.services.google_sheets import GoogleSheetsClient
```

### 2. Dependency Injection Always
No node, worker, or handler instantiates its own dependencies. All clients are built at startup in `dependencies.py` and injected via constructor.

```python
# ✅ Correct
class ExecutionAgentNode:
    def __init__(self, db: DatabaseClient, transport: TransportAdapter): ...

# ❌ Wrong
class ExecutionAgentNode:
    def __init__(self):
        self.db = GoogleSheetsClient()  # hard dependency
```

### 3. Append-Only for User Data
The `DatabaseClient` Protocol must never expose `update_row` or `delete_row` on user persona tabs. Historical data is immutable at the protocol boundary. Schema changes (adding columns) are the only structural mutations allowed.

### 4. No Hardcoded Config
Everything configurable goes through `core/config.py` Pydantic Settings, loaded from `.env`. No strings, IDs, tokens, or URLs hardcoded anywhere in the application.

### 5. Explicit State, No Global Mutations
LangGraph `AgentState` is the single source of truth during a graph run. Nodes read from state, write to state. No global variables, no class-level caches inside nodes.

---

## Error Handling

- Every external API call (Sheets, Discord, Google Drive) must have a typed `try/except` with explicit exception types — never a bare `except: pass`
- 404 from storage backend → trigger re-onboarding flow, not a crash
- Redis unavailable → fail fast with a clear error message on startup, not at request time

---

## Testing Standards

- Every `Protocol` must have a `MockClient` implementation in `services/mock_clients.py`
- Every `feat/` branch must have at least one passing integration test before merge to `dev`
- Tests use mock clients by default — no real API calls in CI
- Test file naming: `test_<module_name>.py` in a `tests/` directory mirroring `src/`

---

## Git Discipline

- Never commit directly to `main`
- Branch naming: `feat/<phase>-<short-description>` (e.g. `feat/phase-0-registry`)
- Commit messages: `type: short description` — types are `feat`, `fix`, `docs`, `chore`, `refactor`, `test`
- Every phase completion gets a SemVer tag on `main`

---

## Naming Conventions

| Thing | Convention | Example |
|:---|:---|:---|
| Files | `snake_case.py` | `schema_inspector.py` |
| Classes | `PascalCase` | `ExecutionAgentNode` |
| Protocol classes | `PascalCase` + suffix `Client` or `Adapter` | `DatabaseClient`, `TransportAdapter` |
| Constants | `UPPER_SNAKE_CASE` | `EXPECTED_SCHEMA_VERSION` |
| LangGraph nodes | `PascalCase` + suffix `Node` | `SemanticGatewayNode` |
| Redis keys | `manas:{scope}:{identifier}` | `manas:channel:in_flight` |
| Env vars | `UPPER_SNAKE_CASE` | `DISCORD_PUBLIC_KEY` |

---

## What Makes a Library vs Application Code?

Extract to a standalone `libs/` package (and eventually a separate repo) when:
- The code has zero knowledge of LangGraph, `AgentState`, or Manas-specific logic
- It would be useful to a developer who has never heard of Manas
- It has a stable Protocol boundary that hasn't changed in 2+ phases
- A second project (e.g. `developer-palette`) also needs to import it

Keep inside `apps/manas-server/` when:
- It references `AgentState`, LangGraph node patterns, or Manas business rules
- It is actively changing with each phase
