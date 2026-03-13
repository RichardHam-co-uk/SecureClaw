# ADR-006: Gateway ↔ Agent IPC Protocol

**Status:** Accepted  
**Date:** March 2026  

---

## Context

The TypeScript Gateway and the Python Agent Runtime run as separate processes and must communicate securely and efficiently. We identified this as an open research gap during planning. The choice of IPC protocol affects security, performance, and maintainability.

## Options Considered

### Option A — Length-Prefixed JSON over Unix Domain Socket
- Simple wire format: 4-byte big-endian length header + UTF-8 JSON body.
- Unix socket: filesystem permission controls access (no network exposure).
- Schema validated with Zod (TS) and Pydantic (Python) on each side.
- Easy to debug (human-readable messages).
- Sufficient for our throughput (single-user Phase 1, small team Phase 2).

### Option B — gRPC over Unix Domain Socket
- Protocol Buffers: strongly typed, compact, fast.
- More complex tooling (protoc, generated stubs in both TS and Python).
- Better for high-throughput Phase 3+ scale.
- Overkill for Phase 1; adds build complexity.

### Option C — REST HTTP over localhost
- Simple, well-understood, easy to debug.
- Binds a localhost TCP port — larger surface area than Unix socket.
- HTTP server in agent adds dependency and attack surface.

### Option D — MessagePack over Unix Domain Socket
- Binary format: faster than JSON, smaller payloads.
- Less human-readable — harder to debug.
- Marginal benefit at our scale.

## Decision

**Phase 1:** Option A — Length-Prefixed JSON over Unix Domain Socket.

**Phase 3 migration path:** If throughput demands it, migrate to Option B (gRPC). The abstract `AgentTransport` interface means this requires only a new transport implementation, not changes to business logic.

## Protocol Specification

```
Frame format:
  [4 bytes: message length, big-endian uint32]
  [N bytes: UTF-8 JSON payload]

Message schema (both directions):
  {
    "version": "1",          // Protocol version — reject mismatches
    "id": "<uuid-v4>",       // Correlation ID for request/response matching
    "type": "request|response|error|ping|pong",
    "tenant_id": "<string>",  // Always present — validated on both sides
    "user_id": "<string>",
    "payload": { ... }        // Type-specific payload, schema-validated
  }

Security rules:
  - tenant_id and user_id validated against session on both sides
  - Unknown message types rejected (no default pass-through)
  - Max message size: 1MB (reject and close connection if exceeded)
  - Ping/pong heartbeat: 30s interval, 5s timeout before reconnect
  - Socket path: /run/secureclaw/agent.sock (mode 0600, owned by secureclaw user)
```

## Consequences

- Simple, auditable wire format.
- Unix socket not exposed on any network interface.
- Both sides must implement schema validation — enforced by SRS R6.1.
- Max message size prevents memory exhaustion attacks.
- Socket file permissions prevent access from other processes on the host.
