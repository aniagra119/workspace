# 0007: Agent Orchestration Tradeoffs & Blob Storage Strategy

**Date:** 2026-05-20
**Status:** Accepted

## Context

After designing the full 9-node LangGraph graph, five orchestration problems and two data architecture questions were identified:

1. Bootstrap nodes are sequential despite being independent.
2. Bootstrap always executes even when the Redis cache is warm.
3. `DatabaseReaderNode` and `DatabaseWriterNode` have a hidden ordering dependency that parallel execution breaks.
4. The full graph is massively overengineered for simple single-intent messages.
5. Lambda cold start can breach Discord's 3-second acknowledgment window before any graph execution begins.

Additionally, two data model questions were open:
- Is a graph database required for cross-persona relationship queries?
- Where does raw blob data (audio, images, documents) live?

## Decisions

### A. Is a Graph Database Required?

**Answer: Not for Phase 1. A `GraphClient` Protocol stub is added for Phase 2.**

The cross-persona correlations in scope (health → finance, mood → productivity) are 1-2 hop relationships. These are solvable with materialized views (`_MANAS_VIEWS`) and tabular aggregations — no graph traversal required.

A graph DB becomes genuinely justified when entity relationships emerge: "John is my roommate and gym partner — show all activities involving John, their cost impact, and how they correlate with my sleep quality." That is a traversal problem across heterogeneous entity types. At that point, a `GraphClient` Protocol is activated.

The `GraphClient` Protocol stub is defined now so the architecture has a clean extension point, but no implementation is required until entity modeling is in scope.

```python
class GraphClient(Protocol):
    """Future: entity-relationship graph for cross-persona traversal queries."""
    async def upsert_node(self, node_id: str, labels: list[str], properties: dict) -> None: ...
    async def upsert_edge(self, from_id: str, to_id: str, rel_type: str, properties: dict) -> None: ...
    async def traverse(self, start_id: str, rel_types: list[str], depth: int) -> list[dict]: ...
```

Candidate backends when needed: **FalkorDB** (Redis-native, serverless-compatible), **Neo4j AuraDB** (managed), **Amazon Neptune** (if already on AWS).

### B. Blob Storage Strategy

Raw binary files (voice memos, images, PDFs, documents) are NOT stored in the vector backend. The pattern is a three-layer separation:

```
Raw file  →  BlobStoreClient (S3 / Google Drive)
                   ↓  extraction (Gemini Vision / Whisper)
Extracted text  →  VectorClient (semantic retrieval)
                   ↓  metadata
blob_ref URL  →  DatabaseClient (column with data_type = blob_ref)
```

The `_MANAS_SCHEMA` `data_type = blob_ref` signals that a column stores a URL pointer, not the content itself. The `DatabaseWriterNode` routes these through the `BlobStoreClient` before writing the URL to the tabular backend.

```python
class BlobStoreClient(Protocol):
    """Upload and retrieve raw binary files."""
    async def upload(self, key: str, data: bytes, content_type: str, metadata: dict) -> str: ...  # returns URL
    async def download(self, url: str) -> bytes: ...
    async def delete(self, url: str) -> None: ...
```

Supported backends: **AWS S3** (primary, pairs with Lambda deployment), **Google Drive** (zero extra cost if already using Sheets). Selection is configurable per persona via `_MANAS_CONFIG`.

### C. Parallel Bootstrap

`SchemaInspectorNode` and `ConfigManagerNode` are independent — they both read from the database and write to disjoint state fields. Running them sequentially doubles cold-path latency for no reason.

**Decision:** Both bootstrap nodes run in parallel using LangGraph's native parallel edge dispatch. The `SemanticGatewayNode` is wired as a `JOIN` node that waits for both to complete before executing.

```
START → [SchemaInspectorNode ∥ ConfigManagerNode] → JOIN → SemanticGatewayNode
```

Cold-path bootstrap latency: halved. Warm-path (Redis hit on both): ~10ms total, both complete near-simultaneously anyway.

### D. Cache-First Dispatch (Warm-Path Optimization)

When the Redis cache is warm, the FastAPI router should short-circuit the bootstrap layer entirely rather than invoking nodes that will just read from cache and return immediately.

The router checks a single Redis key (`manas:{channel_ref}:bootstrap_ready`) before enqueuing the LangGraph job:
- **Cache hit:** enqueue job with `skip_bootstrap=True` flag. The graph starts directly at `SemanticGatewayNode` with pre-populated `active_personas`, `persona_schemas`, `config_rules` from state reconstruction.
- **Cache miss:** run full bootstrap → cognitive layers.

This eliminates two redundant Redis reads per message on the hot path.

### E. Fast Path for Simple Intents

The full 9-node graph is overengineered for high-confidence, single-intent messages like "I spent ₹450 on lunch." These represent the majority of messages. Running bootstrap + semantic gateway + database writer + response synthesizer for a simple row append is wasteful.

**Decision: Two execution paths.**

**Fast Path** (< 200ms target):
- Triggered when the `SemanticGatewayNode` returns a `confidence >= 0.95` single-persona intent with `requires_db_write = True` only.
- Bypasses `ContextFetcherNode`, `DatabaseReaderNode`, `PersonaProvisionerNode`, `InterceptEvaluatorNode`.
- Runs: `SemanticGatewayNode` → `DatabaseWriterNode` → `ResponseSynthesizerNode`.

**Full Path** (500ms–2s):
- All remaining cases: multi-persona routing, context retrieval, schema operations, intercept evaluation, persona provisioning.
- Full 9-node graph executes.

```python
# In SemanticGatewayNode output
class DynamicRouteIntent(BaseModel):
    ...
    confidence: float               # 0.0–1.0 from LLM structured output
    fast_path_eligible: bool        # computed: confidence >= 0.95 and single flag set
```

### F. Hidden Read-Before-Write Dependency

`DatabaseReaderNode` and `DatabaseWriterNode` are currently wired to run in parallel. This breaks the "check before append" pattern — e.g., checking if today's workout is already logged before creating a duplicate row.

**Decision:** The conditional edge logic is updated. `DatabaseWriterNode` waits for `DatabaseReaderNode` when `intent.requires_db_read = True AND intent.requires_db_write = True`. Otherwise they run in parallel.

```
SemanticGatewayNode
    ├── requires_db_read AND requires_db_write → DatabaseReaderNode → DatabaseWriterNode (sequential)
    ├── requires_db_read only               → DatabaseReaderNode (parallel with others)
    └── requires_db_write only              → DatabaseWriterNode (parallel with others)
```

### G. Lambda Cold Start Isolation

The FastAPI HTTP handler is a thin synchronous layer. Its only job is: verify signature, push job to Redis queue, return `{"type": 5}`. This takes < 100ms and is immune to Lambda cold starts because it has no graph initialization.

The LangGraph graph is initialized in a **separate Lambda invocation** triggered by the Redis queue consumer (using Upstash QStash or AWS SQS). Cold start cost is paid by the async worker, not the HTTP handler. The 3-second Discord window is never at risk.

```
HTTP Lambda (always warm, minimal init):  verify → queue → return {"type": 5}
Worker Lambda (cold start acceptable):    dequeue → build_graph() → execute → send_followup
```

## Updated AgentState

```python
class AgentState(TypedDict):
    envelope: TransportEnvelope
    raw_input: str
    active_personas: list[str]
    persona_schemas: dict[str, type]
    config_rules: list[dict]
    db_capabilities: DatabaseCapabilities    # Added: loaded by SchemaInspectorNode
    intent: Optional[Any]                    # DynamicRouteIntent with confidence + fast_path_eligible
    trajectory: Annotated[list[str], operator.add]
    tool_outputs: Annotated[list[dict], operator.add]
    synthesized_response: Optional[str]
    intercept_components: Optional[list[dict]]
```

## Consequences

- Bootstrap latency halved via parallelism (ADR-C).
- Hot-path Redis reads eliminated on warm cache (ADR-D).
- ~70% of messages hit the fast path (single-intent appends) at < 200ms (ADR-E).
- No duplicate row risk from parallel read-write (ADR-F).
- Discord 3-second window is structurally guaranteed regardless of Lambda cold starts (ADR-G).
- `GraphClient` and `BlobStoreClient` are Protocol stubs — zero implementation cost until needed (ADR-A, ADR-B).
- `AgentState.db_capabilities` is now a first-class field read by `DatabaseWriterNode` before structural ops.
