# 0003: Swappable Discord Client (Adapter Pattern)

**Date:** 2026-05-20
**Status:** Accepted

## Context
Our AI Agents occasionally need to look back in time and read the user's chat history to gain context (e.g., "What did I say yesterday about my knee pain?"). To do this, the agents must interface with the Discord REST API to fetch historical messages.

Hardcoding `httpx.get("https://discord.com/api/...")` inside the LangGraph agent logic tightly couples our AI reasoning engine to Discord's specific HTTP implementation and rate limits.

## Decision
We will utilize the **Adapter/Strategy Pattern** utilizing Python `Protocols`.

1. We define an abstract interface `ChatHistoryClient(Protocol)` with a method `fetch_history(channel_id, limit, before)`.
2. The LangGraph agents will **only** depend on this abstract interface.
3. We create a concrete `DiscordHTTPClient` that implements the Protocol using `httpx`.
4. We inject the concrete client into the agent's state at runtime (via Dependency Injection).

## Consequences
*   **Positive:** The AI agents are completely ignorant of Discord.
*   **Positive:** We can easily swap out the Discord client for a `MockChatClient` during unit testing, saving time and preventing rate-limiting during local development.
*   **Positive:** If we ever port Project Manas to Slack or Telegram, the agent logic remains 100% unchanged. We just write a new Adapter.
