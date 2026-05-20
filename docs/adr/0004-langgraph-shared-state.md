# 0004: LangGraph Shared State via Accumulating Reducers

**Date:** 2026-05-20
**Status:** Accepted

## Context

In a multi-node LangGraph graph, nodes execute conditionally and potentially in parallel. The terminal node — `ResponseSynthesizerNode` — must synthesize a grounded reply that reflects everything that occurred during execution: which rows were read from the database, what mutations were queued, what chat history was retrieved, whether an intercept rule fired. Without an explicit sharing mechanism, this context is invisible to the synthesizer.

The naive approach — passing results through explicit edges — does not scale. As the graph grows, every new node requires new edge wiring and new state fields. The synthesizer also cannot distinguish between "nothing happened" and "an error occurred."

## Decision

We use LangGraph's **reducer pattern** (`Annotated[list, operator.add]`) on two fields in `AgentState`:

- `trajectory: Annotated[list[str], operator.add]` — human-readable execution log. Every node appends a structured string: `"[ContextFetcherNode] Retrieved 12 messages from #main-feed before 2026-05-20T09:00:00Z"`.
- `tool_outputs: Annotated[list[dict], operator.add]` — structured raw output. Every node appends a typed dict with its results.

The reducer guarantees that partial state updates from concurrent nodes are merged additively. A node returning `{"trajectory": ["[DatabaseReaderNode] Read 3 rows from health tab"]}` appends to the list — it cannot overwrite another node's entries.

`ResponseSynthesizerNode` is the only terminal node. It reads the complete accumulated `trajectory` and `tool_outputs` and produces `synthesized_response`. It has full causal visibility into every operation that occurred.

## Node Names (Canonical)

| Old Name (deprecated) | Canonical Name |
|:---|:---|
| `MutatorNode` | `DatabaseWriterNode` |
| `DiscordHistoryFetcherNode` | `ContextFetcherNode` |
| `SheetsFetcherNode` | `DatabaseReaderNode` |

All nodes are generic and receive their backend clients via constructor injection.

## Consequences

- Synthesizer is never blind to what the swarm did, including partial failures.
- New nodes added to the graph automatically flow into the synthesizer with zero wiring changes.
- Prompt token cost for the synthesizer grows with trajectory length. Acceptable given Gemini 1.5's context window, and mitigated by trajectory entries being concise structured strings, not raw API responses.
