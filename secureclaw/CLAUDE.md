# SecureClaw — Claude Code Governing Document

## CRITICAL: Read This First

All code generated for this project MUST comply with the Security Requirements Specification (SRS) in `secureclaw/docs/security/SRS.md`. This document governs every module, function, and configuration you generate. Non-compliant code must not be committed.

## Project Overview

SecureClaw is a production-grade, secure-by-design AI agent platform, rebuilt from first principles using OpenClaw as a reference design only. It is **not a fork of OpenClaw**; it shares no direct code lineage.

**Stack:**
- Gateway / Policy Engine / Auth: TypeScript (Node.js 20 LTS)
- Agent Runtime / LLM tooling: Python 3.12
- IPC: Unix domain socket (gRPC-style framing)
- Secrets: HashiCorp Vault (containerised)
- Observability: Prometheus + Loki + Grafana
- Deployment: Docker Compose

## Security Mandates for Code Generation

1. **No plaintext secrets** — ever. All secret references use `SecretRef` descriptors (`env://`, `vault://`).
2. **No arbitrary shell exec** in default/secure mode. All subprocess calls are allowlisted and sandboxed.
3. **All inputs schema-validated** before use (Zod for TypeScript, Pydantic for Python).
4. **All tool calls pass through the Policy Engine** before execution — no exceptions.
5. **All file paths** must be normalised and validated against the tenant workspace root before use.
6. **No network egress from tools** unless explicitly allowlisted in tenant policy.
7. **All audit events** must be emitted via the `AuditLogger` interface — never `console.log`.
8. **No dependencies added** without checking for CVEs via `pnpm audit` / `pip-audit` first.
9. **OIDC/JWT validation** must use a well-audited library (e.g. `jose`) — no custom JWT parsing.
10. **Every module** must have corresponding unit tests covering security-critical paths.

## Module Boundaries

```
secureclaw/
├── gateway/          # TypeScript: Auth, routing, rate-limiting, WebSocket
├── policy/           # TypeScript: Policy engine, RBAC, tool decision
├── agent/            # Python: LLM loop, context assembly, tool orchestration
├── tools/            # Python: Sandboxed tool implementations
├── vault-adapter/    # TypeScript: HashiCorp Vault client abstraction
├── audit/            # TypeScript: Structured audit logger, signed receipts
├── memory/           # Python: Tenant-scoped vector/semantic memory
├── ui/               # TypeScript: Localhost web UI (Lit components)
├── channels/         # TypeScript: Discord, Slack, Teams adapters
├── skills/           # Manifests only — no executable code without approval
├── deploy/           # Docker Compose, Dockerfiles, configs
└── docs/             # Architecture, SRS, runbooks, ADRs
```

## Development Workflow

1. Write/review SRS requirement before implementing any feature.
2. Implement in smallest testable slice.
3. Run security gates: `pnpm audit`, `semgrep`, `pip-audit`, `trivy`.
4. Request Codex security review on every module.
5. All PRs require passing security gate CI before merge.
