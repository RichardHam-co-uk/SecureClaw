# ADR-001: TypeScript Gateway + Python Agent Pattern

**Status:** Accepted  
**Date:** March 2026  

## Context

We need to choose a runtime strategy for SecureClaw. The two primary concerns are: (1) strong typing and security for auth/policy/networking, and (2) access to the Python AI/ML ecosystem for the agent loop.

## Decision

Use TypeScript (Node.js 20 LTS) for the Gateway, Policy Engine, Vault Adapter, and Audit Logger. Use Python 3.12 for the Agent Runtime and Tool layer. The two communicate over a Unix domain socket with a simple length-prefixed JSON framing protocol.

## Rationale

- TypeScript provides compile-time type safety critical for auth and policy logic.
- Node.js has excellent async I/O for a high-concurrency gateway.
- Python has unmatched AI/ML library support (LiteLLM, vector stores, etc.).
- Unix socket IPC is faster than HTTP and does not expose a network port.
- Language boundary creates a natural security boundary: compromise of agent does not give direct gateway access.

## Consequences

- Developers need familiarity with both TypeScript and Python.
- IPC protocol must be versioned and schema-validated on both sides.
- Build pipeline needs both `pnpm` and `uv`/`pip` tooling.
