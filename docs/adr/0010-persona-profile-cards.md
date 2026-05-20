# 0010: Persona Profile Cards and Data Snapshots

**Date:** 2026-05-20
**Status:** Accepted

## Context

As the system collects continuous event logs, the user needs a way to instantly view the "current state" of a persona without issuing a query. 

Two specific requirements emerged:
1. **Live Persona Profile:** A constantly updated summary of the persona's current status (e.g., "Current Weight: 75kg | Last Workout: 2 days ago") maintained as a pinned message in the respective transport channel (e.g., Discord).
2. **Dynamic Channel Metadata:** The bot should be able to update the channel's name and topic/description based on the persona's evolving focus or current trackers.
3. **Snapshots and Filtered Data:** Clarification on whether historical point-in-time snapshots of data are required, and how they are fetched.

## Decisions

### A. Persona Profile Cards and Channel Metadata

Instead of the user asking "what is my current status?", the system will manage the transport channel's UI to reflect the persona's current state.

**Implementation:**
1. **Transport Extension:** The `TransportAdapter` Protocol is extended with `upsert_pinned_status()` and `update_channel()`.
2. **State Tracking:** `_MANAS_CONFIG` is expanded to include a `pinned_message_id` column to allow the system to edit the existing pinned message rather than creating a new one.
3. **Generation:** The Manthan Engine's **Daily Batch Job** will generate a markdown-formatted "Current Status" string based on the latest aggregated data and push it via the `TransportAdapter`. It will also compute a concise channel topic (e.g., "Active trackers: Sleep, Workout | 14-day streak") and invoke `update_channel`.

```python
class TransportAdapter(Protocol):
    ...
    async def update_channel(
        self, channel_ref: str, name: str | None = None, topic: str | None = None
    ) -> None: ...
    async def upsert_pinned_status(
        self, channel_ref: str, content: str, message_id: str | None = None
    ) -> str: ... # Returns the message_id
```

If a user logs a major milestone (e.g., "I just ran 5km"), the fast path or full path can also trigger an immediate profile update if the `SemanticGatewayNode` sets a `requires_profile_update` flag.

### B. Snapshots and Filtered Fetching

**Are snapshots required?** Yes, but they do not require a new architectural paradigm. 
- Point-in-time snapshots are structurally identical to materialized views. 
- A monthly budget snapshot is just a `_MANAS_VIEWS` definition that the Manthan Engine materializes on the 1st of every month. It writes a row to a generic `_SNAPSHOTS` or specialized view tab with the date attached.

**Fetching Filtered Data:** 
- The `DatabaseClient` already supports fetching filtered data: `async def read_rows(self, table_ref: str, filters: dict | None = None) -> list[dict]: ...`
- The `SemanticGatewayNode` generates `intent.filter_params` (e.g., `{"date": ">2026-04-01", "category": "food"}`). The `DatabaseReaderNode` uses this to fetch exact slices of data for the LLM to synthesize.
- No new architecture is needed for filtered data fetching; it is natively supported by the existing Database Protocol.

## Consequences

- The `TransportAdapter` contract now mandates support for pinned/sticky messages. If a transport (like SMS) does not support pins, this fails gracefully (no-op).
- `_MANAS_CONFIG` schema adds `pinned_message_id`.
- The Manthan Engine gains the responsibility of continuously regenerating the Profile Card markdown and syncing it to the transport layer.
