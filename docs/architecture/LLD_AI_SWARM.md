# Exocortex Workspace: Low Level Design (LLD) & AI Swarm

## 1. Discord Webhook Validation (Ed25519)

All incoming traffic from Discord must be cryptographically verified. If an attacker discovers our AWS Lambda API Gateway URL, they could simulate Discord payloads and hijack our AI agents or database.

Discord uses **Ed25519** digital signatures.

### Implementation Details:
The FastAPI `Request` object exposes raw bytes. Before parsing the JSON body, we execute:
```python
from nacl.signing import VerifyKey
from nacl.exceptions import BadSignatureError

def verify_discord_signature(body: bytes, signature: str, timestamp: str, public_key: str) -> bool:
    try:
        verify_key = VerifyKey(bytes.fromhex(public_key))
        verify_key.verify(timestamp.encode() + body, bytes.fromhex(signature))
        return True
    except BadSignatureError:
        return False
```
If this fails, we immediately return `HTTP 401 Unauthorized`.

## 2. LangGraph Multi-Agent Swarm

The core intelligence of Project Manas is built on **LangGraph**, utilizing Gemini as the underlying LLM. We structure this as a deterministic State Machine.

### 2.1 The Global State Schema
State is passed between LangGraph nodes using a strict Pydantic model.

```python
from typing import TypedDict, Annotated, List, Optional
from pydantic import BaseModel, Field

class RouteIntent(BaseModel):
    category: str = Field(description="One of: 'finance', 'health', 'dev', 'meta'")
    extracted_entities: dict = Field(description="Key-value pairs extracted from user input")
    requires_mutation: bool = Field(description="True if Google Sheets must be updated")

class AgentState(TypedDict):
    messages: Annotated[List[str], "The conversation history from Redis"]
    current_input: str
    intent: Optional[RouteIntent]
    database_responses: List[str]
```

### 2.2 Node Orchestration

1.  **`omni_router_node`:** A lightweight LLM call utilizing `gemini-1.5-flash`. Its sole purpose is to populate the `RouteIntent` Pydantic schema using structured output (function calling).
2.  **`data_fetcher_node`:** Triggers only if the intent requires historical context not found in the Redis hot-cache. Queries Google Sheets.
3.  **`mutator_node`:** Constructs the JSON payload representing the cell updates (e.g., appending a row to the Finance tab). It **does not** execute the API call directly; it pushes the payload to the Redis Write Queue.
4.  **`intercept_engine_node`:** The final step. It evaluates the current `_META_STATE` (e.g., `days_since_gym > 2`). If a rule triggers, it modifies the final Discord response to append **UI Message Components** (Interactive Buttons).

## 3. The Interactive Intercepts (Zero-Friction UX)

When the Swarm responds to the user on Discord, it isn't just text. It generates interactive buttons that embed the state directly in the `custom_id` payload.

**Discord JSON Payload Example:**
```json
{
  "content": "Hey, noticed you haven't tracked a workout in 48 hours. Did you hit the gym today?",
  "components": [
    {
      "type": 1,
      "components": [
        {
          "type": 2,
          "label": "Yes, lifted weights",
          "style": 3,
          "custom_id": "log_health_gym_weights_20260520"
        },
        {
          "type": 2,
          "label": "No, rest day",
          "style": 4,
          "custom_id": "log_health_rest_20260520"
        }
      ]
    }
  ]
}
```
When the user taps "Yes, lifted weights", Discord fires a `Type 3` Interaction back to our Webhook. The FastAPI router parses the `custom_id` (`log_health_gym_weights_20260520`), bypasses the LLM Omni-Router entirely, and dumps the pre-formatted data straight into the Redis Write Queue for instantaneous execution.

## 4. Google Sheets API (Cold Ledger) Debouncing

To respect the 60 requests/minute quota of Google Sheets, we employ a debouncing pattern.

1. Agents use the `append_row` command locally, which serializes the action and pushes it to `redis.lpush("sheets_write_queue", payload)`.
2. A separate background worker (or the Manthan Engine loop) runs every 5 minutes.
3. It pops all items from `sheets_write_queue`.
4. It consolidates them into a single `batchUpdate` or `appendCells` Google Sheets API request, utilizing maximum efficiency.
