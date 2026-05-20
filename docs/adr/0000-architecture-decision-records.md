# 0000: Architecture Decision Records (ADRs) Strategy

**Date:** 2026-05-20
**Status:** Accepted

## Context

The Manas ecosystem consists of multiple independent microservices (e.g., `project-manas`, `developer-palette`, and future nodes). Because these are independent codebases but part of a unified ecosystem, documenting cross-cutting concerns (AWS infrastructure, network routing, database architectures, and security policies) inside a single microservice creates a fragmented and confusing source of truth.

## Decision

We will use a **Workspace-Level Meta-Repository** strategy for our architectural documentation.
All ecosystem-wide decisions will be mapped here using Architecture Decision Records (ADRs). 

This `docs/adr/` directory acts as the central brain for the workspace, governing how the independent microservices interact, deploy, and secure themselves.

## Consequences

*   **Positive:** Single source of truth for all cloud/DevOps/Security decisions.
*   **Positive:** Microservices remain 100% independent and portable without dragging along unrelated infrastructure documentation.
*   **Negative:** Developers must remember to check the workspace root for documentation, rather than looking inside the microservice they are currently editing.
