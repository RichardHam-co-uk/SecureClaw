# SecureClaw — Project Plan

**Version:** 1.0  
**Date:** March 2026  
**Owner:** zebadee2kk  
**Branch:** `secureclaw`  
**Tracking:** This document is the canonical progress tracker. Update status fields as work completes.

---

## Vision Summary

Build a production-grade, secure-by-design AI agent platform (SecureClaw) in three phases:

| Phase | Name | Target | Status |
|---|---|---|---|
| 1 | Personal / Homelab POC | Single user, Docker Compose, localhost | 🟡 In Progress |
| 2 | Company / Team Deployment | Multi-tenant, OIDC, Slack/Teams, enterprise audit | ⬜ Not Started |
| 3 | Open Source + Monetisation | Open core release, Enterprise edition, managed service | ⬜ Not Started |

---

## MVP Definition (Phase 1 Complete)

Phase 1 MVP is achieved when ALL of the following are true:

- [ ] User can send a message via CLI and receive an agent response
- [ ] User can send a message via localhost Web UI and receive an agent response
- [ ] All requests require authentication (local signed token)
- [ ] Policy Engine evaluates and logs every tool call before execution
- [ ] At least 3 default tools operational: `read_file`, `web_search`, `structured_query`
- [ ] All tools execute inside isolated Docker containers
- [ ] HashiCorp Vault holds all secrets (no plaintext anywhere)
- [ ] Prompt injection detection active on all inputs (3-layer minimum)
- [ ] Audit log produced for every request (structured, Ed25519 signed)
- [ ] Prometheus metrics and Grafana dashboard operational
- [ ] All CI security gates passing (Semgrep, Snyk, Trivy, pip-audit)
- [ ] Adversarial test suite: 100+ prompt injection cases all blocked
- [ ] `docker compose up` brings up entire stack from cold start in <5 minutes
- [ ] Air-gap deployable (no internet required at runtime)
- [ ] Zero CRITICAL/HIGH CVEs in Snyk/Trivy scan

---

## Stage & Milestone Breakdown

---

### STAGE 0 — Foundation ✅ Complete

**Goal:** Project scaffolding, governance documents, architecture, and CI skeleton in place.  
**Branch:** `secureclaw`  
**Status:** ✅ Done — March 2026

| # | Task | Status |
|---|---|---|
| 0.1 | Fork OpenClaw to zebadee2kk/openclaw | ✅ Done |
| 0.2 | Create `secureclaw` branch | ✅ Done |
| 0.3 | Write Security Requirements Specification (SRS.md) | ✅ Done |
| 0.4 | Write Architecture document + ASCII diagram | ✅ Done |
| 0.5 | Write Threat Model | ✅ Done |
| 0.6 | Write 3 ADRs (gateway/Python, Vault, sandbox) | ✅ Done |
| 0.7 | Write VISION.md (phases + monetisation) | ✅ Done |
| 0.8 | Write SECURITY.md (disclosure policy) | ✅ Done |
| 0.9 | Write CLAUDE.md (governing doc for Claude Code) | ✅ Done |
| 0.10 | Write CODEX_REVIEW_CHECKLIST.md | ✅ Done |
| 0.11 | Write Claude Code prompt templates | ✅ Done |
| 0.12 | Write Incident Response runbook | ✅ Done |
| 0.13 | Write Docker Compose stack skeleton | ✅ Done |
| 0.14 | Write Dockerfile.gateway + Dockerfile.agent (multi-stage) | ✅ Done |
| 0.15 | Write GitHub Actions security gates CI workflow | ✅ Done |
| 0.16 | Write this project plan | ✅ Done |

---

### STAGE 1 — Core Infrastructure

**Goal:** Working TypeScript gateway with auth + Python agent runtime skeleton. Services boot and communicate.  
**Target Completion:** 4 weeks from start  
**Status:** ⬜ Not Started

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

**Milestone 1.1 Exit Criteria:** Gateway boots, rejects unauthenticated requests, validates schemas, rate limits.

#### Milestone 1.2 — Vault Integration

| # | Task | Owner | Status |
|---|---|---|---|
| 1.2.1 | Scaffold `secureclaw/vault-adapter/` TypeScript module | Claude Code | ⬜ |
| 1.2.2 | Implement `SecretRef` descriptor type (`env://`, `vault://`) | Claude Code | ⬜ |
| 1.2.3 | Implement Vault client (short-lived tokens, auto-renewal) | Claude Code | ⬜ |
| 1.2.4 | Implement secret resolution at startup (zero-on-use) | Claude Code | ⬜ |
| 1.2.5 | Write Vault init/unseal runbook | Manual | ⬜ |
| 1.2.6 | Configure Vault container in Compose (dev mode → prod notes) | Claude Code | ⬜ |
| 1.2.7 | Write first-run script: Vault init, policy setup, seed secrets | Claude Code | ⬜ |
| 1.2.8 | Codex security review: vault adapter | Codex | ⬜ |

**Milestone 1.2 Exit Criteria:** `docker compose up` starts Vault; gateway resolves `SecretRef` at boot; no plaintext secrets in any file.

#### Milestone 1.3 — Agent Runtime Skeleton

| # | Task | Owner | Status |
|---|---|---|---|
| 1.3.1 | Scaffold `secureclaw/agent/` Python project (uv, pyproject.toml) | Claude Code | ⬜ |
| 1.3.2 | Implement Unix socket IPC server (agent side) | Claude Code | ⬜ |
| 1.3.3 | Implement Unix socket IPC client (gateway side) | Claude Code | ⬜ |
| 1.3.4 | Implement IPC protocol: length-prefixed JSON, versioned schema | Claude Code | ⬜ |
| 1.3.5 | Implement LiteLLM adapter: multi-provider (Anthropic, OpenAI, Ollama) | Claude Code | ⬜ |
| 1.3.6 | Implement provider config loading via SecretRef | Claude Code | ⬜ |
| 1.3.7 | Implement basic agent loop (single turn, no tools yet) | Claude Code | ⬜ |
| 1.3.8 | Implement max-iteration and timeout guards | Claude Code | ⬜ |
| 1.3.9 | Unit tests: IPC framing, LLM adapter mocking, loop guards | Claude Code | ⬜ |
| 1.3.10 | Codex security review: agent module | Codex | ⬜ |

**Milestone 1.3 Exit Criteria:** End-to-end: CLI message → gateway → agent → LLM → response back to CLI. No tools yet.

---

### STAGE 2 — Security Core

**Goal:** Policy Engine, Prompt Injection Detection, and Audit Logger operational. Security model enforced end-to-end.  
**Target Completion:** 4 weeks after Stage 1  
**Status:** ⬜ Not Started

#### Milestone 2.1 — Policy Engine

| # | Task | Owner | Status |
|---|---|---|---|
| 2.1.1 | Scaffold `secureclaw/policy/` TypeScript module | Claude Code | ⬜ |
| 2.1.2 | Implement `PolicyDecision` type | Claude Code | ⬜ |
| 2.1.3 | Implement RBAC: roles (viewer/analyst/operator/admin), tool mappings | Claude Code | ⬜ |
| 2.1.4 | Implement `decide()` function | Claude Code | ⬜ |
| 2.1.5 | Implement secure default policy (all dangerous tools off) | Claude Code | ⬜ |
| 2.1.6 | Implement per-tenant policy override loading | Claude Code | ⬜ |
| 2.1.7 | Wire Policy Engine into agent tool call path | Claude Code | ⬜ |
| 2.1.8 | Unit tests: allow, deny, role escalation attempts, override | Claude Code | ⬜ |
| 2.1.9 | Codex security review: policy module | Codex | ⬜ |

**Milestone 2.1 Exit Criteria:** No tool call reaches execution without Policy Engine approval. Denied calls logged.

#### Milestone 2.2 — Prompt Injection Detection

| # | Task | Owner | Status |
|---|---|---|---|
| 2.2.1 | Implement Layer 1: static regex/semantic pre-pass | Claude Code | ⬜ |
| 2.2.2 | Build injection pattern library (20+ patterns minimum) | Claude Code | ⬜ |
| 2.2.3 | Implement Layer 3: CanaryManager (inject + monitor) | Claude Code | ⬜ |
| 2.2.4 | Implement Layer 4: OutputGuard (validate LLM output) | Claude Code | ⬜ |
| 2.2.5 | Implement session quarantine on canary trigger | Claude Code | ⬜ |
| 2.2.6 | Write adversarial test suite: 100+ prompt injection cases | Claude Code | ⬜ |
| 2.2.7 | Validate all 100+ cases blocked and logged | Testing | ⬜ |
| 2.2.8 | Codex security review: injection detection | Codex | ⬜ |

**Milestone 2.2 Exit Criteria:** All adversarial test cases blocked. Canary triggers quarantine. OutputGuard prevents malformed tool calls.

#### Milestone 2.3 — Audit Logger

| # | Task | Owner | Status |
|---|---|---|---|
| 2.3.1 | Scaffold `secureclaw/audit/` TypeScript module | Claude Code | ⬜ |
| 2.3.2 | Implement `AuditEvent` Zod schema | Claude Code | ⬜ |
| 2.3.3 | Implement `SecretRedactor` (regex-based, configurable) | Claude Code | ⬜ |
| 2.3.4 | Implement Ed25519 signer (key from Vault) | Claude Code | ⬜ |
| 2.3.5 | Implement Loki transport (hot tier) | Claude Code | ⬜ |
| 2.3.6 | Implement local file transport (warm tier) | Claude Code | ⬜ |
| 2.3.7 | Instrument all audit points: auth, policy, tool, LLM, error | Claude Code | ⬜ |
| 2.3.8 | Unit tests: redaction, signature verify, tamper detect | Claude Code | ⬜ |
| 2.3.9 | Codex security review: audit module | Codex | ⬜ |

**Milestone 2.3 Exit Criteria:** Every request produces a signed audit event. Tamper detection works. Redaction verified.

---

### STAGE 3 — Tool Layer

**Goal:** Default tool set operational inside sandboxed containers. Tool policy enforced.  
**Target Completion:** 3 weeks after Stage 2  
**Status:** ⬜ Not Started

#### Milestone 3.1 — Tool Sandbox Infrastructure

| # | Task | Owner | Status |
|---|---|---|---|
| 3.1.1 | Build `secureclaw/deploy/Dockerfile.sandbox` (minimal, non-root) | Claude Code | ⬜ |
| 3.1.2 | Implement sandbox spawn/destroy lifecycle manager | Claude Code | ⬜ |
| 3.1.3 | Implement workspace bind-mount scoping (path normalisation) | Claude Code | ⬜ |
| 3.1.4 | Implement container capability drop + seccomp profile | Claude Code | ⬜ |
| 3.1.5 | Implement pre-warmed container pool (reduce latency) | Claude Code | ⬜ |
| 3.1.6 | Test: path traversal attempts blocked at mount level | Testing | ⬜ |
| 3.1.7 | Trivy scan sandbox image | CI | ⬜ |

**Milestone 3.1 Exit Criteria:** Tool containers spawn and destroy cleanly. Path traversal impossible. Trivy clean.

#### Milestone 3.2 — Default Tools

| # | Task | Owner | Status |
|---|---|---|---|
| 3.2.1 | Implement `read_file` tool (workspace-scoped, read-only) | Claude Code | ⬜ |
| 3.2.2 | Implement `web_search` tool (via allowlisted search API) | Claude Code | ⬜ |
| 3.2.3 | Implement `structured_query` tool (read-only memory) | Claude Code | ⬜ |
| 3.2.4 | Implement tool result type + validation | Claude Code | ⬜ |
| 3.2.5 | Implement egress deny-by-default + domain allowlist | Claude Code | ⬜ |
| 3.2.6 | Unit tests: each tool with valid + invalid inputs | Claude Code | ⬜ |
| 3.2.7 | Integration test: full tool call chain (policy → sandbox → result) | Testing | ⬜ |
| 3.2.8 | Codex security review: tool layer | Codex | ⬜ |

**Milestone 3.2 Exit Criteria:** All 3 default tools operational, sandbox-isolated, policy-gated.

#### Milestone 3.3 — Memory Layer

| # | Task | Owner | Status |
|---|---|---|---|
| 3.3.1 | Scaffold `secureclaw/memory/` Python module | Claude Code | ⬜ |
| 3.3.2 | Implement tenant-scoped vector store (ChromaDB or similar) | Claude Code | ⬜ |
| 3.3.3 | Implement workspace `.md` file indexing (OpenClaw-compatible) | Claude Code | ⬜ |
| 3.3.4 | Implement memory encryption at rest | Claude Code | ⬜ |
| 3.3.5 | Implement query isolation: tenant_id scoped at query layer | Claude Code | ⬜ |
| 3.3.6 | Codex security review: memory module | Codex | ⬜ |

**Milestone 3.3 Exit Criteria:** Memory persists across sessions. Cross-tenant query impossible. Encrypted at rest.

---

### STAGE 4 — Observability & Hardening

**Goal:** Full monitoring stack, anomaly alerting, incident response tested. Production-ready posture.  
**Target Completion:** 2 weeks after Stage 3  
**Status:** ⬜ Not Started

#### Milestone 4.1 — Observability Stack

| # | Task | Owner | Status |
|---|---|---|---|
| 4.1.1 | Configure Prometheus scrape targets for all services | Claude Code | ⬜ |
| 4.1.2 | Implement custom metrics: request rate, policy denials, tool usage, LLM latency | Claude Code | ⬜ |
| 4.1.3 | Configure Loki log ingestion for audit logs | Claude Code | ⬜ |
| 4.1.4 | Build Grafana dashboard: SecureClaw security overview | Claude Code | ⬜ |
| 4.1.5 | Configure Grafana dashboard: policy denial alerts | Claude Code | ⬜ |
| 4.1.6 | Implement anomaly alerting: webhook to configurable target | Claude Code | ⬜ |

**Milestone 4.1 Exit Criteria:** `docker compose up` brings up full observability. Dashboard shows live metrics.

#### Milestone 4.2 — Security Hardening Pass

| # | Task | Owner | Status |
|---|---|---|---|
| 4.2.1 | Full Semgrep + Snyk + Trivy pass on all modules | CI | ⬜ |
| 4.2.2 | Config lint: verify no insecure defaults ship | CI | ⬜ |
| 4.2.3 | Full adversarial test run (100+ cases) | Testing | ⬜ |
| 4.2.4 | Manual penetration test: prompt injection, path traversal, SSRF, auth bypass | Manual | ⬜ |
| 4.2.5 | Remediate all findings | Manual | ⬜ |
| 4.2.6 | Document residual risks in THREAT_MODEL.md | Manual | ⬜ |
| 4.2.7 | Full Codex review of complete codebase | Codex | ⬜ |

**Milestone 4.2 Exit Criteria:** Zero CRITICAL/HIGH CVEs. Pentest complete. All findings remediated or risk-accepted.

#### Milestone 4.3 — First Run & Developer Experience

| # | Task | Owner | Status |
|---|---|---|---|
| 4.3.1 | Write `secureclaw/deploy/first-run.sh`: Vault init, secrets seeding, compose up | Claude Code | ⬜ |
| 4.3.2 | Write `docs/deployment/QUICKSTART.md`: install to first agent response in <15 steps | Manual | ⬜ |
| 4.3.3 | Write `docs/deployment/HOMELAB.md`: Proxmox/Docker deployment guide | Manual | ⬜ |
| 4.3.4 | Write `docs/deployment/SECURE_CHECKLIST.md`: pre-production checklist | Manual | ⬜ |
| 4.3.5 | Test cold-start: `docker compose up` to working agent in <5 min | Testing | ⬜ |

**Milestone 4.3 Exit Criteria:** A new user can follow QUICKSTART.md and have a working, secure SecureClaw in under 15 minutes.

---

### STAGE 5 — MVP Validation & Phase 1 Sign-off

**Goal:** All MVP criteria met. Phase 1 formally complete. Ready to start Phase 2 planning.  
**Target Completion:** 1 week after Stage 4  
**Status:** ⬜ Not Started

| # | Task | Owner | Status |
|---|---|---|---|
| 5.1 | Run full MVP checklist (see above) | Manual | ⬜ |
| 5.2 | Run full CI pipeline clean | CI | ⬜ |
| 5.3 | Sign-off security review (self-audit against SRS) | Manual | ⬜ |
| 5.4 | Tag release `v0.1.0-mvp` | Manual | ⬜ |
| 5.5 | Write `CHANGELOG.md` entry for v0.1.0 | Manual | ⬜ |
| 5.6 | Update VISION.md: Phase 1 complete, Phase 2 begins | Manual | ⬜ |
| 5.7 | Draft Phase 2 plan (team deployment, OIDC, Slack/Teams) | Perplexity + zebadee2kk | ⬜ |

**Milestone 5 Exit Criteria:** `v0.1.0-mvp` tag exists. All MVP criteria green. Phase 2 plan drafted.

---

## Phase 2 Preview — Team / Enterprise Deployment

*(Detailed plan drafted at Stage 5 sign-off)*

| Stage | Focus | Key Deliverables |
|---|---|---|
| 6 | OIDC / Entra ID auth | Replace local token with full OIDC, M365/Entra integration |
| 7 | Multi-tenant model | Tenant provisioning, isolation, per-tenant config and billing hooks |
| 8 | Channel integrations | Slack, Microsoft Teams, Discord adapters with per-channel policy |
| 9 | Enterprise audit | Cold tier (S3), SIEM export, SOC 2 readiness review |
| 10 | Opt-in tools | `write_file`, `http_client`, `script_runner` with elevated policy |
| 11 | Phase 2 validation | Security review, penetration test, internal company deployment |

---

## Phase 3 Preview — Open Source + Monetisation

*(Detailed plan drafted at Phase 2 sign-off)*

| Stage | Focus | Key Deliverables |
|---|---|---|
| 12 | Open core preparation | Licence selection, community edition scope, enterprise feature split |
| 13 | Enterprise edition | SSO federation, SIEM connectors, SLA framework, managed Vault |
| 14 | Public release | README, docs site, security advisory, launch comms |
| 15 | Managed service | Hosted offering design, pricing, billing integration |
| 16 | Consulting productisation | Service catalogue using cybersecurity expertise |

---

## Overall Timeline (Estimated)

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

> Timeline assumes part-time development (evenings/weekends) using Claude Code for implementation and Codex for review. Adjust based on available hours per week.

---

## Tools & Workflow

| Tool | Role |
|---|---|
| **Claude Code** | Primary implementation — use prompts from `docs/development/CLAUDE_CODE_PROMPTS.md` |
| **Codex** | Security review — use `docs/development/CODEX_REVIEW_CHECKLIST.md` on every module |
| **Perplexity** | Research, planning, architecture decisions, this document |
| **GitHub Actions** | CI security gates — automated on every PR/push to `secureclaw` branch |
| **VS Code** | Primary IDE |
| **Docker Compose** | Local development and homelab deployment |

---

## Progress Update Log

| Date | Stage | Update | Author |
|---|---|---|---|
| March 2026 | Stage 0 | Foundation complete — all docs, architecture, CI, Dockerfiles committed | zebadee2kk |

---

## How to Update This Document

1. When a task completes, change `⬜` to `✅`.
2. When a milestone is complete, update its **Exit Criteria** status and set the stage **Status** to `✅ Done`.
3. Add a row to the **Progress Update Log** with date, stage, and summary.
4. When Phase 1 MVP criteria are all green, tag `v0.1.0-mvp` and begin Phase 2 planning.
