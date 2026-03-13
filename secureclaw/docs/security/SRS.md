# SecureClaw Security Requirements Specification (SRS)

**Version:** 1.0  
**Date:** March 2026  
**Status:** Active — Governing Document  
**Owner:** zebadee2kk  

---

## 1. Purpose

This document governs all code generation (Claude Code), security review (Codex/Semgrep), and testing for the SecureClaw project. Any code, configuration, or deployment artefact that contradicts a requirement in this document must be rejected and remediated before merge.

---

## 2. Threat Model

### 2.1 Threat Actors

| Actor | Motivation | Entry Points |
|---|---|---|
| External attacker | Unauthorised access, data exfil, RCE | Public endpoints, exposed ports |
| Compromised user | Privilege escalation, cross-tenant access | Auth layer, session tokens |
| Prompt attacker | Bypass policy, exfil data via LLM | Message inputs, skill inputs |
| Malicious skill/extension | Code execution, persistence | Skill loader, extension system |
| Insider / misconfiguration | Weaken defaults, expose internals | Config files, admin API |
| Supply chain | Backdoor via deps/images | Package manager, container registry |

### 2.2 Attack Surface

- Network endpoints (gateway WebSocket, admin API, UI)
- Configuration and secrets loading at startup
- Tool execution layer (filesystem, network, subprocess)
- LLM prompt construction and response parsing
- Memory/query layer (vector DB, workspace files)
- Skill/extension loading
- Container and host interface

---

## 3. Security Requirements

### R1 — Authentication & Authorisation

- **R1.1** All external endpoints require authentication. No unauthenticated access to any endpoint in any mode.
- **R1.2** Supported auth providers: OIDC/JWT (Entra ID, generic OIDC) for enterprise; local signed token for homelab/dev. Switching requires config change only, not code change.
- **R1.3** RBAC with defined roles: `viewer`, `analyst`, `operator`, `admin`. Tools are mapped to minimum required role.
- **R1.4** Session tokens are scoped to `tenant_id + user_id`, expire in ≤24h, and are rotated on logout or anomaly detection.
- **R1.5** Admin/destructive operations require step-up MFA (TOTP minimum; FIDO2/YubiKey preferred).
- **R1.6** `tenant_id` must be present and validated on every request, at every layer. No implicit "default" tenant bypass.

### R2 — Network Security

- **R2.1** Default bind: `127.0.0.1:8080` only. Remote binding requires explicit `SECURECLAW_HOST` env var and TLS configuration.
- **R2.2** TLS 1.3 minimum when remote binding is enabled. No self-signed certs in production.
- **R2.3** All tool network egress is deny-by-default. Explicit per-tenant domain/port allowlists required. No wildcard allowlists.
- **R2.4** Rate limiting on all inbound endpoints: per-IP and per-user, configurable per tenant.
- **R2.5** CORS: strict origin allowlist, no wildcard origins.

### R3 — Sandboxing & Privilege Separation

- **R3.1** Gateway runs as non-root user (UID 1000), no unnecessary Linux capabilities, read-only root filesystem except for designated write paths.
- **R3.2** Each tool invocation spawns in an isolated container (Docker/Podman rootless), with: dedicated non-root OS user, chroot/bind-mount to tenant workspace subtree only, `--no-new-privileges`, dropped all capabilities except `DAC_READ_SEARCH` where needed, seccomp profile blocking dangerous syscalls.
- **R3.3** File operations: paths normalised (resolve symlinks, strip `..`), validated against tenant workspace root before use. Absolute paths outside workspace are rejected.
- **R3.4** No arbitrary shell/exec in secure mode. A restricted `ScriptRunner` tool (opt-in, `admin` role only) executes only pre-approved script templates.
- **R3.5** Python agent runtime and TypeScript gateway communicate only over a Unix domain socket. No other inter-process communication.

### R4 — Secrets Management

- **R4.1** No plaintext secrets in any config file, environment file committed to version control, or log output.
- **R4.2** All secret references use `SecretRef` descriptors: `env://VAR_NAME`, `vault://secret/path`. Resolved at runtime only.
- **R4.3** HashiCorp Vault (containerised) is the primary secrets backend. Service accounts use short-lived Vault tokens only — no static API keys.
- **R4.4** Human access to Vault requires TOTP MFA minimum. Privileged paths (rotate/delete) require FIDO2/hardware token.
- **R4.5** LLM prompt construction and tool argument logging must redact any resolved secret values. Regex-based redaction applied at the audit logger boundary.
- **R4.6** API key rotation triggered on: scheduled interval (configurable, default 90d), manual request, or anomaly detection.

### R5 — Policy Engine & Guardrails

- **R5.1** A central Policy Engine evaluates every tool call before execution. Signature: `decide(user, tenant, role, tool_name, tool_args, context) → {allow: bool, reason: string}`.
- **R5.2** Secure defaults: `exec`, `browser`, `write_file`, `http_client` are disabled in default profile. Opt-in requires explicit config and minimum `operator` role.
- **R5.3** Per-tenant tool allowlists and blocklists. Per-channel role overrides supported.
- **R5.4** Cross-tenant data access is architecturally prevented: all DB queries, file paths, and memory lookups are scoped by `tenant_id` at the query layer.
- **R5.5** Policy decisions are logged as structured audit events regardless of allow/deny outcome.

### R6 — Input Validation & LLM Safety

- **R6.1** All inbound data (HTTP bodies, WebSocket messages, tool arguments) validated against Zod (TypeScript) or Pydantic (Python) schemas. Unvalidated data must not reach business logic.
- **R6.2** Prompt injection detection is multi-layer:
  - Layer 1: Static pre-pass — regex and semantic pattern matching on raw input before LLM sees it.
  - Layer 2: Attention Tracker pattern — monitor for instruction-hijack signatures at inference time.
  - Layer 3: Canary tokens — per-session canary strings embedded in system prompt; appearance in tool args triggers immediate session quarantine.
  - Layer 4: Output guard — all LLM outputs parsed and validated before tool calls are constructed.
- **R6.3** Skills are local-only, carry GPG-signed manifests, and require explicit human approval before installation. No runtime remote fetch of skill code.
- **R6.4** LLM jailbreak attempts detected via output guard → session flagged → human-in-the-loop review triggered.

### R7 — Auditing & Observability

- **R7.1** Structured audit log emitted for every: auth attempt, policy decision, tool invocation (args redacted per R4.5), LLM call (token counts only), anomaly detection event, error.
- **R7.2** Audit log entries are Ed25519-signed at write time for tamper-evidence.
- **R7.3** Tiered retention: Hot (0–30d, Loki), Warm (30–365d, compressed encrypted local), Cold (1–7yr, optional S3-compatible with object lock).
- **R7.4** Metrics exported to Prometheus: request rate, tool usage, policy denials, error rate, LLM latency.
- **R7.5** Anomaly alerting: policy denial threshold breach, canary token trigger, repeated auth failures → webhook to configurable target.

### R8 — Deployment & Supply Chain

- **R8.1** Multi-stage Docker builds: build stage (full deps), runtime stage (distroless or minimal Alpine), non-root user, signed images.
- **R8.2** CI pipeline security gates (must pass before merge): Semgrep SAST, Snyk SCA, Trivy container scan, pip-audit, adversarial prompt test suite, config linter.
- **R8.3** Config linter rejects: `auth=none`, `bind=0.0.0.0` without TLS, `sandbox=false`, wildcard egress domains.
- **R8.4** All dependencies pinned to exact versions. Dependabot/Renovate enabled for automated PR updates with security context.
- **R8.5** Air-gap deployable: all runtime dependencies bundled in image; no internet required at runtime.

---

## 4. OWASP Alignment

| OWASP LLM Top 10 | SecureClaw Mitigation |
|---|---|
| LLM01 Prompt Injection | R6.2 multi-layer detection, canary tokens |
| LLM02 Insecure Output Handling | R6.4 output guard, schema validation |
| LLM03 Training Data Poisoning | Scoped memory, tenant isolation (R5.4) |
| LLM04 Model Denial of Service | Rate limiting (R2.4), token count limits |
| LLM05 Supply Chain Vulnerabilities | R8.2 SCA, pinned deps, signed images |
| LLM06 Sensitive Info Disclosure | R4.5 secret redaction, R5.4 tenant isolation |
| LLM07 Insecure Plugin Design | R6.3 signed skills, policy engine (R5.1) |
| LLM08 Excessive Agency | R5.2 secure defaults, R3.2 sandboxing |
| LLM09 Overreliance | Human-in-the-loop gates, audit trail (R7) |
| LLM10 Model Theft | Auth on all endpoints (R1.1), no model exposure |

---

## 5. Compliance Targets

- OWASP Secure-by-Design Framework
- OWASP LLM Top 10
- NIST CSF 2.0 (Identify, Protect, Detect, Respond, Recover)
- CIS Docker Benchmark v1.6
- SOC 2 Type II readiness (audit retention R7.3)

---

## 6. Enforcement

- Claude Code must reference this document in every session before generating code.
- Codex security review is required on every module and every PR.
- No merge without all CI security gates green (R8.2).
- SRS amendments require explicit approval and version bump.
