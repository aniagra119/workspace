# 0002: Dynamic Pydantic Schema Generation

**Date:** 2026-05-20
**Status:** Accepted

## Context
Traditionally, when utilizing LLMs with structured outputs (like Gemini's Function Calling), developers must define static schemas (e.g., hardcoded Pydantic classes) in the codebase. However, Project Manas maps unstructured data into Google Sheets. If a user adds a new column to their Google Sheet, the hardcoded Pydantic model in the Python code becomes immediately obsolete, requiring a new code deployment to support the new column.

## Decision
We will dynamically instantiate Pydantic models at runtime based on the Google Sheet's structure.

1. The system fetches the column headers from the active Google Sheet tab.
2. It uses `pydantic.create_model` to generate a dynamic class (e.g., `DynamicRowEntry`) on the fly, where each sheet column maps to a Pydantic field.
3. This dynamically generated class is passed to `gemini.with_structured_output()`.

## Consequences
*   **Positive:** Complete decoupling of the AI schema from the codebase.
*   **Positive:** Zero-deployment updates. Users can literally add a column to their spreadsheet (e.g., "Calories") and the AI will immediately start extracting that data in the next chat.
*   **Negative:** Increased latency. We must fetch the Google Sheet headers *before* we can construct the LLM prompt. (This is mitigated by caching the headers in Redis).
