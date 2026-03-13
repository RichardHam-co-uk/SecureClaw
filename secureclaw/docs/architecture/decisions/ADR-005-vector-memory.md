# ADR-005: Vector Database — Qdrant over ChromaDB

**Status:** Accepted  
**Date:** March 2026  

---

## Context

The SecureClaw memory layer requires a vector database for semantic search over workspace files and conversation history. The original plan referenced ChromaDB as a candidate. Security research performed during planning reveals critical concerns with ChromaDB for our use case.

## ChromaDB Assessment

**Security concerns (disqualifying):**
- No authentication out of the box in any mode.
- Tenant isolation relies entirely on application-level filtering — there is no database-level enforcement. A bug in our query layer could expose cross-tenant data.
- No access control at the collection level.
- Does not align with SRS R5.4 (cross-tenant data access must be architecturally prevented at the query layer).

## Qdrant Assessment

**Security strengths:**
- Collection-per-tenant isolation: each tenant gets a dedicated collection with its own access control.
- API key authentication supported natively.
- TLS support for client connections.
- Runs cleanly as a Docker container with persistent volume.
- Active security maintenance and CVE response track record.
- Read-only API key support — tools that only need `structured_query` get a read-only key.

**Performance:** Qdrant benchmarks significantly ahead of ChromaDB on recall and latency at our target scale.

## Decision

Use **Qdrant** as the vector database for the SecureClaw memory layer.

## Tenant Isolation Model

**Phase 1 (single tenant, low scale):** One Qdrant collection per tenant. Tenant ID is part of the collection name (`sc_{tenant_id}_workspace`, `sc_{tenant_id}_history`). This provides maximum isolation.

**Phase 3 (many enterprise tenants):** Switch to partition-key isolation within shared collections. The memory module interface abstracts this — switching requires only a config change and data migration, not code change.

## Implementation Notes

- Qdrant container added to `docker-compose.yml` on the `backend` network (internal only).
- Two API keys provisioned at first-run: `read-write` (agent context assembly) and `read-only` (tool `structured_query`).
- Both keys stored in Vault, referenced via `SecretRef`.
- Collections named `sc_{tenant_id}_{purpose}` — tenant_id validated before collection name is constructed (prevent injection).
- All queries include explicit tenant collection scoping — no cross-collection queries permitted.

## Consequences

- Removes ChromaDB from the stack entirely.
- Qdrant added to `docker-compose.yml`.
- Memory module implementation targets Qdrant Python client.
- `DEPENDENCIES.md` updated with Qdrant version pin.
- ADR supersedes any ChromaDB references in previous documents.
