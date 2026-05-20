# 0001: AWS Deployment Strategy for the AI Swarm

**Date:** 2026-05-20
**Status:** ~~Proposed~~ → **SUPERSEDED by ADR-0012**
**Superseded by:** [ADR-0012: Self-Hosted Docker Worker Architecture](0012-self-hosted-docker-architecture.md)

---

This ADR originally proposed a **Hybrid Serverless (Lambda-First)** architecture using AWS Lambda for the FastAPI webhook router and LangGraph swarm.

That decision has been **superseded**. See ADR-0012 for the current deployment model.

**Summary of why Lambda was rejected:**
- Lambda's 15-minute execution timeout conflicts with persistent Redis workers that must run continuous polling loops (the `agent_worker.py` and `write_debouncer.py` processes).
- Lambda is completely stateless — every invocation is cold. The LangGraph `AgentState` cannot persist across Lambda calls without serializing the entire state to Redis on every node boundary, which adds significant latency and complexity.
- Persistent Docker containers avoid all cold-start penalties and enable native async Redis queue polling without any workarounds.
- The self-hosted model eliminates AWS costs entirely for personal/development use — Docker runs locally for free.

The original decision matrix (Lambda vs ECS) is preserved below for historical reference.

---

## Original Decision Matrix (Archived)

| Criteria | AWS Lambda (Serverless) | AWS ECS Fargate (Containers) |
| :--- | :--- | :--- |
| **Startup / Scale** | Near-instant scaling from zero to thousands. | Slower spin-up times (minutes). Requires minimum instances. |
| **Cost Profile** | Pay per millisecond of execution. $0 when idle. | Pay for provisioned vCPU/RAM uptime. Baseline cost even when idle. |
| **Timeout Limits** | Hard 15-minute execution limit. | Unlimited execution time. |
| **Discord Webhooks** | Perfect for handling sporadic bursts of incoming Discord interactions. | Requires maintaining a warm HTTP pool to avoid cold-start timeouts. |
| **Statefulness** | Completely stateless. Requires Upstash/Redis for shared memory. | Can run stateful background loops (Manthan Engine) natively. |
| **Packaging (uv/Docker)** | Max 10GB container image. `uv` dependencies are perfectly compatible. | First-class Docker support. No size limits. |
