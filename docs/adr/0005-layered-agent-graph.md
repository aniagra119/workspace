# 0005: Transport Envelope + Client Injection (Full Decoupling)

**Date:** 2026-05-20
**Status:** Accepted

## Context

Three problems were identified during architecture review of the initial LangGraph design:

**1. Transport coupling in cognitive state.**
`AgentState` contained `channel_id: str`, `user_id: str`, `discord_response: str` — Discord-specific fields embedded in a schema that should be platform-agnostic. This directly violated ADR-0003, which mandated swappable transport clients.

**2. Transport-coupled node naming.**
`DiscordHistoryFetcherNode` named after a concrete transport, even though ADR-0003 had already established a `ChatHistoryClient` Protocol. The node was not actually using the Protocol as its interface — it was importing Discord-specific HTTPX logic directly.

**3. Missing meta-management layer.**
The graph had no node responsible for structural database operations (create tab, add column). It also had no node for reading `_MANAS_CONFIG` or `_MANAS_SCHEMA`. The `SemanticGatewayNode` was assumed to have `active_personas` already in state with no mechanism to populate them.

## Decision

### A. Transport Envelope

A `TransportEnvelope` dataclass carries all transport-specific metadata. It is populated by the FastAPI router before the graph is invoked and is carried as an opaque field in `AgentState`. No cognitive node reads or writes it except `ResponseSynthesizerNode`, which passes it to the `TransportAdapter` for final delivery.

This means all cognitive nodes — `SemanticGatewayNode`, `DatabaseReaderNode`, `ContextFetcherNode`, etc. — contain zero import statements referencing Discord, Slack, or any transport library.

### B. Protocol-Based Client Injection

Three Protocols define the boundaries between the graph and external systems:

- `ChatHistoryClient` — fetches messages from any conversational platform
- `DatabaseClient` — reads and writes to any tabular database
- `TransportAdapter` — sends messages and provisions channels

Concrete implementations (`DiscordHistoryClient`, `GoogleSheetsClient`, `DiscordAdapter`) are wired in `src/api/dependencies.py` and injected into nodes at graph construction time. Swapping a backend = one line change in `dependencies.py`.

Nodes are named for their function, not their current backend:
- `ContextFetcherNode` (not `DiscordHistoryFetcherNode`)
- `DatabaseReaderNode` (not `SheetsFetcherNode`)
- `DatabaseWriterNode` (not `MutatorNode`)

### C. Bootstrap Layer + Meta-Management

Two new always-running cached nodes form the bootstrap layer:
- `SchemaInspectorNode`: reads `_MANAS_SCHEMA` + persona tab headers, builds `persona_schemas` via `pydantic.create_model`, writes `active_personas`. Redis TTL cached.
- `ConfigManagerNode`: reads `_MANAS_CONFIG`, writes `config_rules`. Redis TTL cached.

`DatabaseWriterNode` handles both row-level mutations and structural operations (create tab, add column, rename column), distinguished by a `MutationIntent.operation_type` field. This makes persona provisioning a first-class graph operation rather than out-of-band logic.

## Consequences

- Any cognitive node can be unit-tested by injecting `MockDatabaseClient` and `MockHistoryClient` from `src/services/mock_clients.py`.
- Adding Slack support requires writing `SlackAdapter` and `SlackHistoryClient` and changing two lines in `dependencies.py`.
- Schema evolution (user adding a column to their sheet) is handled entirely in the bootstrap layer without code changes — `SchemaInspectorNode` detects the new header and updates `persona_schemas` on the next Redis TTL expiry.
