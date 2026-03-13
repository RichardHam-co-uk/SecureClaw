# SecureClaw — Project Plan

**Version:** 1.1  
**Date:** March 2026  
**Owner:** zebadee2kk  
**Branch:** `secureclaw`  
**Tracking:** This document is the canonical progress tracker. Update status fields as work completes.

---

## Changelog

| Version | Date | Changes |
|---|---|---|
| 1.0 | March 2026 | Initial plan |
| 1.1 | March 2026 | ChromaDB → Qdrant (ADR-005); Vault Transit auto-unseal added (ADR-004); IPC protocol resolved (ADR-006); Audit signing clarified (ADR-007); Canary token design locked (ADR-008); DEPENDENCIES.md added with all pinned versions and excluded packages |

---

## Vision Summary

| Phase | Name | Target | Status |
|---|---|---|---|
| 1 | Personal / Homelab POC | Single user, Docker Compose, localhost | 🟡 In Progress |
| 2 | Company / Team Deployment | Multi-tenant, OIDC, Slack/Teams, enterprise audit | ⬜ Not Started |
| 3 | Open Source + Monetisation | Open core release, Enterprise edition, managed service | ⬜ Not Started |

---

## Research Gaps Resolved

All open research gaps from the planning phase have been resolved and documented as ADRs:

| Gap | Resolution | ADR |
|---|---|---|
| IPC protocol (gateway ↔ agent) | Length-prefixed JSON over Unix socket | ADR-006 |
| Seccomp profile strategy | Docker default + custom deny-list for tools | Stage 3.1 task |
| Ed25519 key management | Vault Transit secrets engine — key never leaves Vault | ADR-007 |
| Canary token format | HMAC-SHA256-derived, session-scoped, innocuous format | ADR-008 |
| Vector DB tenant isolation | Qdrant (replaces ChromaDB), collection-per-tenant | ADR-005 |
| Vault auto-unseal homelab | Transit auto-unseal (secondary Vault container) | ADR-004 |
| OpenClaw licence | Research pending — check before Phase 3 open-source release | Deferred |

---

## MVP Definition (Phase 1 Complete)

Phase 1 MVP is achieved when ALL of the following are true:

- [ ] User can send a message via CLI and receive an agent response
- [ ] User can send a message via localhost Web UI and receive an agent response
- [ ] All requests require authentication (local signed token)
- [ ] Policy Engine evaluates and logs every tool call before execution
- [ ] At least 3 default tools operational: `read_file`, `web_search`, `structured_query`
- [ ] All tools execute inside isolated Docker containers
- [ ] HashiCorp Vault holds all secrets; Transit auto-unseal operational
- [ ] Prompt injection detection active (4-layer: static, canary, attention, output guard)
- [ ] Audit log produced for every request (structured, Ed25519-signed via Vault Transit)
- [ ] Prometheus metrics and Grafana dashboard operational
- [ ] All CI security gates passing (Semgrep, Snyk, Trivy, pip-audit, config lint)
- [ ] Adversarial test suite: 100+ prompt injection cases all blocked
- [ ] `docker compose up` brings up entire stack from cold start in <5 minutes
- [ ] Air-gap deployable (no internet required at runtime)
- [ ] Zero CRITICAL/HIGH CVEs in Snyk/Trivy scan
- [ ] All dependencies at or above minimum safe versions in DEPENDENCIES.md

---

## Full Stack Components

| Component | Technology | Purpose | ADR |
|---|---|---|---|
| Gateway | TypeScript / Node.js 20 LTS | Auth, rate-limit, schema validation, channel adapters | ADR-001 |
| Policy Engine | TypeScript | RBAC, tool policy, prompt guard | — |
| Agent Runtime | Python 3.12 | LLM loop, context assembly, canary, output guard | ADR-001 |
| LLM Layer | LiteLLM ≥1.82.1 | Multi-provider abstraction (Anthropic, OpenAI, Ollama) | — |
| Tool Sandbox | Docker/Podman | Per-invocation isolated containers | ADR-003 |
| Secrets | HashiCorp Vault 1.18.3 | All credentials, canary secret, session signing key | ADR-002 |
| Vault Transit Unseal | HashiCorp Vault (secondary) | Auto-unseal for primary Vault | ADR-004 |
| Vector Memory | Qdrant 1.13.2 | Tenant-scoped semantic memory, workspace indexing | ADR-005 |
| Audit Logger | TypeScript + Vault Transit | Ed25519-signed structured events | ADR-007 |
| IPC | Unix socket, length-prefix JSON | Gateway ↔ Agent communication | ADR-006 |
| Canary Tokens | Python (HMAC-SHA256) | Session-scoped prompt injection detection | ADR-008 |
| Logs | Grafana Loki 3.4.2 | Hot-tier audit + application logs | — |
| Metrics | Prometheus 3.2.1 | Service metrics, policy denial counters | — |
| Dashboards | Grafana 11.6.1 | Security overview, alerting | — |

---

## Stage & Milestone Breakdown

### STAGE 0 — Foundation ✅ Complete

| # | Task | Status |
|---|---|---|
| 0.1–0.16 | All foundation tasks (docs, architecture, CI, Dockerfiles, plan) | ✅ Done |
| 0.17 | ADR-004: Vault Transit auto-unseal | ✅ Done |
| 0.18 | ADR-005: Qdrant over ChromaDB | ✅ Done |
| 0.19 | ADR-006: IPC protocol (length-prefix JSON / Unix socket) | ✅ Done |
| 0.20 | ADR-007: Vault Transit audit signing | ✅ Done |
| 0.21 | ADR-008: HMAC-SHA256 canary tokens | ✅ Done |
| 0.22 | DEPENDENCIES.md: all packages pinned with CVE dates | ✅ Done |
| 0.23 | VAULT_SETUP.md: full init runbook | ✅ Done |

---

### STAGE 1 — Core Infrastructure
**Target Completion:** 4 weeks from start | **Status:** ⬜ Not Started

#### Milestone 1.1 — Gateway Skeleton
| # | Task | Owner | Status |
|---|---|---|---|
| 1.1.1 | Scaffold `secureclaw/gateway/` TypeScript project (pnpm, tsconfig, tsdown) | Claude Code | ⬜ |
| 1.1.2 | Implement `auth/` module: JWT validation (jose), local token validation | Claude Code | ⬜ |
| 1.1.3 | Implement session type: tenant_id, user_id, role, expiry | Claude Code | ⬜ |
| 1.1.4 | Implement rate limiter middleware (per-IP, per-user, sliding window) | Claude Code | ⬜ |
| 1.1.5 | Implement Zod schema validation middleware | Claude Code | ⬜ |
| 1.1.6 | Implement CLI channel adapter | Claude Code | ⬜ |
| 1.1.7 | Implement WebSocket channel adapter (localhost UI) | Claude Code | ⬜ |
| 1.1.8 | Implement `/health` endpoint | Claude Code | ⬜ |
| 1.1.9 | Unit tests: auth paths, rate limiting, schema validation | Claude Code | ⬜ |
| 1.1.10 | Codex security review: auth module | Codex | ⬜ |
| 1.1.11 | Semgrep clean on gateway code | CI | ⬜ |

**Exit Criteria:** Gateway boots, rejects unauthenticated requests, validates schemas, rate limits.

#### Milestone 1.2 — Vault Integration
| # | Task | Owner | Status |
|---|---|---|---|
| 1.2.1 | Scaffold `secureclaw/vault-adapter/` TypeScript module | Claude Code | ⬜ |
| 1.2.2 | Implement `SecretRef` descriptor type (`env://`, `vault://`) | Claude Code | ⬜ |
| 1.2.3 | Implement Vault client (short-lived tokens, auto-renewal) | Claude Code | ⬜ |
| 1.2.4 | Implement secret resolution at startup (zero-on-use) | Claude Code | ⬜ |
| 1.2.5 | Add `vault-transit` service to docker-compose.yml | Claude Code | ⬜ |
| 1.2.6 | Add `qdrant` service to docker-compose.yml | Claude Code | ⬜ |
| 1.2.7 | Write first-run.sh: Vault init, Transit setup, policies, secret seeding | Claude Code | ⬜ |
| 1.2.8 | Verify VAULT_SETUP.md runbook against first-run.sh | Manual | ⬜ |
| 1.2.9 | Codex security review: vault adapter | Codex | ⬜ |

**Exit Criteria:** `docker compose up` starts Vault with Transit auto-unseal; gateway resolves SecretRef at boot; no plaintext secrets in any file.

#### Milestone 1.3 — Agent Runtime Skeleton
| # | Task | Owner | Status |
|---|---|---|---|
| 1.3.1 | Scaffold `secureclaw/agent/` Python project (uv, pyproject.toml) | Claude Code | ⬜ |
| 1.3.2 | Implement Unix socket IPC server (agent side, length-prefix JSON) | Claude Code | ⬜ |
| 1.3.3 | Implement Unix socket IPC client (gateway side) | Claude Code | ⬜ |
| 1.3.4 | Implement IPC protocol schema (Zod + Pydantic, versioned) | Claude Code | ⬜ |
| 1.3.5 | Implement LiteLLM adapter: multi-provider (litellm ≥1.82.1) | Claude Code | ⬜ |
| 1.3.6 | Implement provider config loading via SecretRef | Claude Code | ⬜ |
| 1.3.7 | Implement basic agent loop (single turn, no tools yet) | Claude Code | ⬜ |
| 1.3.8 | Implement max-iteration and timeout guards | Claude Code | ⬜ |
| 1.3.9 | Unit tests: IPC framing, LLM adapter mocking, loop guards | Claude Code | ⬜ |
| 1.3.10 | Codex security review: agent module | Codex | ⬜ |

**Exit Criteria:** End-to-end: CLI message → gateway → Unix socket → agent → LLM → response back to CLI.

---

### STAGE 2 — Security Core
**Target Completion:** 4 weeks after Stage 1 | **Status:** ⬜ Not Started

#### Milestone 2.1 — Policy Engine
| # | Task | Owner | Status |
|---|---|---|---|
| 2.1.1–2.1.9 | (as per v1.0) | Claude Code / Codex | ⬜ |

#### Milestone 2.2 — Prompt Injection Detection
| # | Task | Owner | Status |
|---|---|---|---|
| 2.2.1 | Implement Layer 1: static regex/semantic pre-pass | Claude Code | ⬜ |
| 2.2.2 | Build injection pattern library (20+ patterns minimum) | Claude Code | ⬜ |
| 2.2.3 | Implement CanaryManager using HMAC-SHA256 design (ADR-008) | Claude Code | ⬜ |
| 2.2.4 | Implement Layer 4: OutputGuard (validate LLM output) | Claude Code | ⬜ |
| 2.2.5 | Implement session quarantine on canary trigger | Claude Code | ⬜ |
| 2.2.6 | Write adversarial test suite: 100+ prompt injection cases | Claude Code | ⬜ |
| 2.2.7 | Validate all 100+ cases blocked and logged | Testing | ⬜ |
| 2.2.8 | Codex security review: injection detection | Codex | ⬜ |

#### Milestone 2.3 — Audit Logger
| # | Task | Owner | Status |
|---|---|---|---|
| 2.3.1–2.3.9 | (as per v1.0, Vault Transit signing per ADR-007) | Claude Code / Codex | ⬜ |

---

### STAGE 3 — Tool Layer
**Target Completion:** 3 weeks after Stage 2 | **Status:** ⬜ Not Started

#### Milestone 3.1–3.3 — Tool Sandbox, Default Tools, Memory Layer (Qdrant)
| Notes | Replaces ChromaDB with Qdrant per ADR-005. Collection-per-tenant model. |
|---|---|
| 3.3.2 | Implement Qdrant collection-per-tenant model (`sc_{tenant_id}_workspace`) | Claude Code | ⬜ |
| 3.3.3 | Implement workspace `.md` file indexing | Claude Code | ⬜ |
| 3.3.4 | Implement encryption at rest (Qdrant volume encryption) | Claude Code | ⬜ |
| 3.3.5 | Read-only API key for `structured_query` tool (from Vault) | Claude Code | ⬜ |

---

### STAGE 4 — Observability & Hardening
**Target Completion:** 2 weeks after Stage 3 | **Status:** ⬜ Not Started

| (Tasks as per v1.0) | | | |
|---|---|---|---|

---

### STAGE 5 — MVP Validation & Phase 1 Sign-off
**Target Completion:** 1 week after Stage 4 | **Status:** ⬜ Not Started

| (Tasks as per v1.0, updated MVP checklist) | | |
|---|---|---|

---

## Overall Timeline

| Phase | Stage | Duration | Cumulative |
|---|---|---|---|
| Phase 1 | Stage 0 — Foundation | ✅ Done | Week 0 |
| Phase 1 | Stage 1 — Core Infrastructure | 4 weeks | Week 4 |
| Phase 1 | Stage 2 — Security Core | 4 weeks | Week 8 |
| Phase 1 | Stage 3 — Tool Layer | 3 weeks | Week 11 |
| Phase 1 | Stage 4 — Observability & Hardening | 2 weeks | Week 13 |
| Phase 1 | Stage 5 — MVP Validation | 1 week | Week 14 |
| Phase 2 | Stages 6–11 | 10–12 weeks | Week 26 |
| Phase 3 | Stages 12–16 | TBD | TBD |

---

## Progress Update Log

| Date | Stage | Update | Author |
|---|---|---|---|
| March 2026 | Stage 0 | Foundation complete (v1.0): all docs, architecture, CI, Dockerfiles committed | zebadee2kk |
| March 2026 | Stage 0 | v1.1 update: 5 new ADRs, DEPENDENCIES.md, VAULT_SETUP.md, ChromaDB → Qdrant, all research gaps resolved | zebadee2kk |
