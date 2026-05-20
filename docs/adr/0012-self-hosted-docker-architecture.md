# 0012: Self-Hosted Docker Worker Architecture

**Date:** 2026-05-20
**Status:** Accepted
**Replaces:** [ADR-0001: AWS Deployment Strategy](0001-aws-deployment-strategy.md)

---

## Context

The original plan (ADR-0001) proposed AWS Lambda as the deployment target. After deeper analysis of the LangGraph execution model, Redis queue polling, and the write debouncer requirements, Lambda was rejected.

## Decision

The system is deployed as a **self-hosted persistent Docker Compose stack**. There is no serverless infrastructure in the Phase 1 design.

```
docker-compose.yml
├── manas-server    # FastAPI webhook router (uvicorn, port 9000)
├── agent-worker    # Persistent Redis queue consumer (LangGraph swarm)
├── write-debouncer # Persistent write coalescer (Google Sheets batch writer)
└── redis           # Local Redis instance (can be swapped for Upstash in prod)
```

## Consequences

**Positive:**
- Zero cold starts. The agent worker polls Redis continuously and can begin processing a job in < 50ms.
- Persistent workers can maintain long-running polling loops without hitting any timeout limits.
- `docker compose up -d` is a single command to spin up the entire system locally.
- Zero cloud cost for personal/development use.

**Negative:**
- Requires a machine to be running 24/7 for production use (a cheap VPS, Raspberry Pi, or a cloud VM). Not truly serverless.
- Horizontal scaling requires Docker Swarm or Kubernetes — not needed for Phase 1.

## Phase 3 Path (Managed SaaS)

When multi-tenant SaaS is required (Phase 3), the architecture graduates to:
- **ECS Fargate** or **Fly.io** for container orchestration (replaces local Docker Compose)
- **Upstash Redis** replaces the local Redis container
- A **VPC perimeter** with an edge Auth service for JWT validation (see Security section in `project-manas/docs/ARCHITECTURE.md`)
