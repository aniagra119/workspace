# 0008: Bidirectional Sync, Message Coalescing, AI Self-Optimization, and App Artifacts

**Date:** 2026-05-20
**Status:** Accepted

## Context

Five related requirements extend the system beyond a one-directional Discord → Sheets flow:

1. The user can edit their Google Sheets database directly (mobile/desktop). These edits should invalidate the system's Redis cache and optionally trigger confirmations back.
2. During long async processing, new messages from the same channel should be held and coalesced into the running job rather than spawning a competing parallel execution.
3. The AI should track its own operational performance and propose self-improvements.
4. The Manthan Engine should generate low-friction user-facing artifacts (dashboards, reports, simple web apps) from sheet data as low-hanging fruit suggestions.
5. The system's AI memory — which personas are active, what data was accessed, what strategies worked — should be persisted and used to improve future decisions.

## Decisions

### A. Two-Way Sync — Sheets as an Input Source

**Problem:** A user edits a cell in the Health tab on their phone. The Redis cache for `health` is now stale. The system has no idea.

**Solution: Apps Script onChange webhook.**

An Apps Script bound to the user's spreadsheet fires on every edit:

```javascript
// Bound Apps Script in the user's Google Spreadsheet
function onEdit(e) {
  const sheet = e.range.getSheet();
  if (sheet.getName().startsWith('_MANAS')) return; // ignore system tabs
  
  UrlFetchApp.fetch('https://<lambda-url>/webhook/sheets-edit', {
    method: 'POST',
    contentType: 'application/json',
    headers: { 'X-Manas-Secret': PropertiesService.getScriptProperties().getProperty('WEBHOOK_SECRET') },
    payload: JSON.stringify({
      persona_id: sheet.getName(),
      row: e.range.getRow(),
      column: e.range.getColumn(),
      new_value: e.value,
      old_value: e.oldValue,
      edited_by: 'user',
    })
  });
}
```

The FastAPI router handles `POST /webhook/sheets-edit` with a shared secret (not Ed25519 — no Discord SDK involved). This populates a `TransportEnvelope` with `source = "sheets_edit"`. The resulting graph execution:

1. `SchemaInspectorNode` — invalidates only the affected persona's cache
2. Skips all cognitive nodes (no LLM inference needed for a direct cell edit)
3. Optionally sends a Discord confirmation if the edit crosses an intercept threshold

**Two-way tracker:** when `status` column in a tracker row is flipped directly in the sheet (e.g. habit checked off), the onChange fires, the cache invalidates, and the Manthan Engine detects the change on its next cycle. No LLM call. Instant.

**Apps Script vs. Manthan Engine boundary:**
- Apps Script: ONLY the onChange event bridge. Nothing more. Its quota limitations (6 min/run, 90 min/day) make it unsuitable for any computation.
- Manthan Engine (scheduled Lambda): all LLM analysis, tracker proposals, correlation detection, artifact generation.
- Native Sheets formulas (QUERY, SUMIF, ARRAYFORMULA): static computed views that update automatically without any Lambda invocation. These are in `_MANAS_VIEWS` with `view_type = formula`.

---

### B. Message Coalescing (Per-Channel In-Flight Lock)

**Problem:** The user sends three rapid messages to `#health-coach` while a job is processing. Without coordination, three parallel graph executions run, each blind to the others, potentially writing three duplicate rows.

**Solution: Per-channel in-flight lock + message buffer.**

```
redis key: manas:{channel_ref}:in_flight     → job_id (string, TTL=120s)
redis key: manas:{channel_ref}:msg_buffer    → list of serialized message payloads
```

The FastAPI webhook handler logic:

```python
async def handle_discord_interaction(envelope: TransportEnvelope, payload: dict):
    lock_key = f"manas:{envelope.channel_ref}:in_flight"
    buffer_key = f"manas:{envelope.channel_ref}:msg_buffer"

    if await redis.get(lock_key):
        # Job in flight — buffer this message, return deferred response immediately
        await redis.rpush(buffer_key, serialize(payload))
        return {"type": 5}  # acknowledged, will handle after current job

    # No job in flight — set lock and enqueue
    job_id = str(uuid4())
    await redis.set(lock_key, job_id, ex=120)
    await redis.lpush("manas:job_queue", serialize({"job_id": job_id, "payload": payload}))
    return {"type": 5}
```

When a Worker Lambda completes a job:

```python
async def on_job_complete(job_id: str, channel_ref: str, result: AgentState):
    # Clear the lock
    await redis.delete(f"manas:{channel_ref}:in_flight")

    # Drain the buffer
    buffered = await redis.lrange(f"manas:{channel_ref}:msg_buffer", 0, -1)
    await redis.delete(f"manas:{channel_ref}:msg_buffer")

    if buffered:
        # Coalesce all buffered messages into a single new job
        # SemanticGatewayNode receives them as a list and processes holistically
        coalesced_payload = {
            "messages": [deserialize(m) for m in buffered],
            "context": result.tool_outputs,  # Previous job's outputs as context
        }
        new_job_id = str(uuid4())
        await redis.set(f"manas:{channel_ref}:in_flight", new_job_id, ex=120)
        await redis.lpush("manas:job_queue", serialize({"job_id": new_job_id, "payload": coalesced_payload}))
```

The `SemanticGatewayNode` is aware that it may receive a `messages: list` rather than a single `raw_input`. When coalesced, it processes all messages as a unified intent batch and returns a single `synthesized_response` covering all of them. The user receives one coherent reply, not three fragmented ones.

---

### C. AI Self-Optimization — Operational Telemetry

The system accumulates operational knowledge in a `_MANAS_META` system tab and a Redis telemetry store. The Manthan Engine reads this data on its analysis cycle and proposes improvements via Discord.

#### `_MANAS_META` — Operational Telemetry Tab

| Column | Type | Description |
|:---|:---|:---|
| `ts` | `datetime` | Timestamp of the event |
| `event_type` | `enum` | `node_execution` \| `extraction_result` \| `rule_fire` \| `tracker_activation` \| `schema_drift` |
| `persona_id` | `str` | Affected persona |
| `node_name` | `str` | Which node |
| `metric_key` | `str` | e.g. `extraction_confidence`, `execution_ms`, `null_rate` |
| `metric_value` | `float` | Numeric value |
| `metadata` | `json` | Free-form context |

The `ResponseSynthesizerNode` writes a telemetry batch to `_MANAS_META` after every execution. Writes are debounced via Redis (batched every N minutes to avoid quota exhaustion).

#### What the Manthan Engine learns:

```python
SELF_OPTIMIZATION_SIGNALS = {
    "column_null_rate":     "Column X has 80% null rate → propose removal or config change",
    "rule_false_positive":  "Rule Y fires but user never completes the prompted action → tune threshold",
    "peak_traffic_hours":   "Persona Z gets 3x messages 8-10pm → shift reminder windows",
    "persona_imbalance":    "Finance persona has 90% of all messages → suggest topic splitting",
    "slow_node":            "DatabaseReaderNode P95 = 2.4s on health tab → cache read results",
    "tracker_abandonment":  "Tracker T has not received data in 14 days → propose deprecation",
    "schema_drift":         "New column detected in Health tab not in _MANAS_SCHEMA → propose registration",
}
```

The Manthan Engine surfaces these as Discord proposals: "Your 'Mood' column has a 73% null rate. Should I make it optional, remove it, or change the reminder wording?" User responds via Component button.

#### AI Memory — Operational Context Store

Beyond telemetry, the AI accumulates a structured memory of its own decisions:

```python
# Stored in Redis + persisted to _MANAS_META periodically
AI_MEMORY = {
    "persona_access_frequency": {"health": 0.42, "finance": 0.38, "dev": 0.20},
    "successful_extraction_patterns": {"finance": ["spent X on Y", "paid X for Y", "X USD on Y"]},
    "failed_extraction_patterns": {"health": ["feeling a bit off"]},  # too vague to extract
    "db_strategy_performance": {"GoogleSheetsClient": {"p50_ms": 180, "p95_ms": 890}},
    "user_preference_signals": {"prefers_brief_responses": True, "timezone": "Asia/Kolkata"},
}
```

This memory is injected into `SemanticGatewayNode`'s system prompt as context, improving extraction accuracy and routing decisions over time without requiring any prompt engineering from the user.

---

### D. AI-Generated App Artifacts

The Manthan Engine can propose and generate low-friction user-facing artifacts from sheet data. These are flagged as suggestions and require one-tap user approval via Discord Component.

**Tier 1 — Zero infrastructure (Sheets-native):**
- QUERY()/ARRAYFORMULA computed views in `_MANAS_VIEWS`
- Google Sheets Charts embedded in a dedicated "Dashboard" tab
- Conditional formatting rules applied programmatically via the Sheets API

**Tier 2 — No-code tools (zero hosting cost):**
- **Google Looker Studio dashboard** — auto-generated data source URL connecting to the user's sheet. The AI generates the JSON config and sends the user a setup link.
- **Google Sites page** — pulls live data from the sheet. Zero hosting. The AI generates the page structure and data range references.

**Tier 3 — Generated web apps (Lambda-hosted):**
- A simple HTML/CSS/JS dashboard generated by the AI, hosted on the same Lambda as the FastAPI app (static file serving via `StaticFiles`). Reads directly from the Sheets API. Updated on each Manthan Engine cycle.

**Artifact proposal flow:**
1. Manthan Engine detects: "User has 90 days of Finance data. A monthly spend breakdown dashboard is feasible."
2. Posts to Discord: "I can build you a budget dashboard. [📊 Preview] [✅ Build It] [❌ Not Now]"
3. User taps "Build It" → Component handler triggers `ArtifactGeneratorNode` (new node, runs outside the main graph as a background job).
4. Artifact is generated, URL is sent to Discord.
5. `_MANAS_VIEWS` is updated with the artifact reference.

---

### E. Schema Independence Confirmation

The four system tabs (`_MANAS_CONFIG`, `_MANAS_SCHEMA`, `_MANAS_TRACKERS`, `_MANAS_VIEWS`) and `_MANAS_META` are logical data contracts. They have no Discord or Google Sheets dependency:
- "Tab" maps to "table" in PostgreSQL, "database" in Notion, "collection" in MongoDB.
- The `DatabaseClient` Protocol's `get_headers / read_rows / create_table` methods abstract all of this.
- The `SchemaInspectorNode` reads these tables via the `DatabaseClient` regardless of backend — zero Discord imports in any of these paths.

The AI's ability to suggest and create things from data is entirely a function of the Manthan Engine's analysis quality, not the backend. The same analysis runs whether the data is in Sheets, Notion, or PostgreSQL.

## Consequences

- A new `POST /webhook/sheets-edit` endpoint is needed with shared-secret verification (not Ed25519).
- `AgentState.raw_input` becomes `raw_inputs: list[str]` to handle coalesced message batches.
- `_MANAS_META` is the fifth system-owned tab — `SchemaInspectorNode` and `ConfigManagerNode` skip it (it is write-only from the graph's perspective; only the Manthan Engine reads it).
- The Manthan Engine becomes a first-class scheduled worker with its own Lambda, cron schedule, and IAM role — not just an ad-hoc analysis concept.
- A new `ArtifactGeneratorNode` runs as an out-of-band background job (not part of the main LangGraph graph).
- `SemanticGatewayNode` prompt gains AI Memory context injection.
