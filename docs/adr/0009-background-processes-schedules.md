# 0009: Background Processes and Cron Schedule Mapping

**Date:** 2026-05-20
**Status:** Accepted

## Context

With the introduction of the Manthan Engine, write debouncing, and message coalescing, Project Manas relies on several background processes operating outside the synchronous Discord/FastAPI hot path. To ensure system stability, avoid Google Sheets API quota exhaustion, and manage Lambda compute costs, we must rigorously map every asynchronous and scheduled process.

The system needs to map:
1. Continuous queue consumers (Write Debouncer, LangGraph Worker)
2. Scheduled Manthan Engine sub-processes (Telemetry, Memory, Trackers, Views)

## Decisions

### 1. Process Segmentation by Frequency

We divide background tasks into three distinct tiers:
- **Tier 1: Event-Driven Queue Consumers** (Continuous / sub-second trigger)
- **Tier 2: Fast Cron Jobs** (Every 15-60 minutes)
- **Tier 3: Daily Batch Jobs** (Once per day, deep analysis)

### 2. Event-Driven Queue Consumers

These are triggered by AWS SQS or Upstash QStash webhook pushes. They scale to zero and run only when data is in the queue.

| Process | Trigger | Estimated Execution Time | Function |
|:---|:---|:---|:---|
| **Agent Worker** | Redis `manas:job_queue` | 500ms – 2.5s | Executes the LangGraph graph. Reads from cached state, makes LLM calls, routes intent. |
| **Write Debouncer** | Redis `manas:write_queue` | 200ms – 500ms | Drains pending `append_row`, `create_table`, and `add_column` jobs. Coalesces them into bulk `batchUpdate` Sheets API calls. Helps avoid the 60 requests/minute Sheets quota limit. |
| **Coalesced Job Runner** | End of Agent Worker | 500ms – 2.5s | Drains `msg_buffer` for a channel if messages arrived while the previous job was in-flight. |

### 3. Manthan Engine Scheduled Jobs (Cron)

The Manthan Engine is not a single monolith; it's a suite of jobs with varying computational costs.

| Sub-Process | Cron Schedule | Est. Execution Time | Function |
|:---|:---|:---|:---|
| **View Materializer** | `*/15 * * * *` (Every 15 min) | 1s - 5s | Scans `_MANAS_VIEWS` where `view_type = materialized`. Re-aggregates data from source tabs and writes static snapshots. Skipped if no new rows exist. |
| **Telemetry & Memory Sync** | `0 * * * *` (Every 1 hour) | 2s - 4s | Reads `_MANAS_META` debounce buffer in Redis. Computes updated `AI_MEMORY`. Persists both back to Google Sheets. |
| **Tracker & Correlation Proposer** | `0 23 * * *` (Daily 23:00) | 10s - 30s | Deep LLM analysis over the past 24 hours of data. Mines cross-persona patterns, proposes new trackers, generates artifact suggestions. Pushes Discord Component prompts. |
| **Schema Migration Runner** | Manual / As Needed | < 5s | Processes `SchemaMigrationJob` from Redis if `EXPECTED_SCHEMA_VERSION` mismatched. |

### 4. Concurrency & Quota Protection

- **Google Sheets API:** The Write Debouncer is rate-limited to execute no more than once per second, coalescing up to 100 rows per `batchUpdate`.
- **LLM Rate Limits:** The Daily Batch Job (Tracker Proposer) uses a `sleep()` throttle between API calls to avoid bursting the Gemini API limit when analyzing multiple personas.
- **Cost Control:** By isolating the deep analysis (Correlation/Trackers) to a daily 23:00 run, we prevent unbounded token usage. The fast cron jobs (View Materialization, Telemetry) use pure Python aggregation, zero LLM calls.

## Consequences

- The Manthan Engine Lambda requires an Amazon EventBridge (or Upstash QStash) configuration with three separate cron triggers calling three different entrypoints (`/manthan/views`, `/manthan/telemetry`, `/manthan/daily-analysis`).
- `AI_MEMORY` updates lag real-time events by up to 1 hour, which is perfectly acceptable for system-level behavioral tuning.
- Google Sheets quotas are heavily protected by the Write Debouncer; peak user traffic will never crash the system.
