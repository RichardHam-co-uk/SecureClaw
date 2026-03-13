# ADR-003: Docker Container per Tool Invocation

**Status:** Accepted  
**Date:** March 2026  

## Context

OpenClaw executes tools as direct Node.js function calls within the gateway process, running as the same OS user. This allows any tool bug or malicious input to compromise the entire process and host filesystem.

## Decision

Each tool invocation spawns a short-lived Docker/Podman container with: a dedicated non-root OS user, bind-mount limited to the tenant workspace subtree, `--no-new-privileges`, all capabilities dropped (re-added only as explicitly required), a seccomp profile, and network access only if the tool requires it (and only to allowlisted destinations).

## Rationale

- Container isolation limits blast radius of tool compromise to the container.
- Bind-mount scoping prevents filesystem traversal outside workspace.
- Short-lived containers (destroyed after invocation) prevent persistence.
- Aligns with R3.2 of the SRS.

## Consequences

- Tool invocation latency increased by container startup (~100–300ms). Mitigated by pre-warmed container pools for default tools.
- Increased Docker daemon load. Monitor and tune pool size.
- Container image for sandbox must be minimal and regularly scanned (Trivy).
