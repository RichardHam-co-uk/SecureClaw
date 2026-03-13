# SecureClaw Threat Model

**Version:** 1.0  
**Date:** March 2026  

---

## 1. System Assets

| Asset | Sensitivity | Notes |
|---|---|---|
| User messages and context | High | May contain PII, credentials, business data |
| LLM API keys | Critical | Compromise = financial and data risk |
| Tenant workspace files | High | May contain source code, configs, secrets |
| Vault root token | Critical | Full secrets access |
| Audit log signing key | High | Compromise = ability to forge audit trail |
| Session tokens | High | Account takeover risk |
| Agent memory/vector store | Medium-High | Could contain sensitive extracted data |

---

## 2. Trust Boundaries

```
[External User] --(TLS + Auth)--> [Gateway] --(validated)--> [Policy Engine]
[Policy Engine] --(approved)--> [Agent Runtime] --(IPC)--> [Tool Sandbox]
[Tool Sandbox] --(scoped)--> [Tenant Workspace] (read/write)
[Agent Runtime] --(SecretRef)--> [Vault] (secrets, MFA-protected)
[All services] --(write-only)--> [Audit Logger]
```

No component has more trust than its position in the chain above. The Gateway does not trust the Agent; the Agent does not trust the Tool; the Tool does not trust the filesystem beyond its mount.

---

## 3. Threat Scenarios

### T1 — Prompt Injection via User Input
**Attack:** User sends crafted message designed to override system prompt and cause agent to exfil data or execute unauthorised tools.  
**Mitigations:** R6.2 (multi-layer detection), canary tokens, output guard, policy engine re-check before tool execution.  
**Residual risk:** Low — four independent layers must all fail.

### T2 — Unauthenticated Endpoint Access
**Attack:** Attacker discovers gateway on network and sends requests without auth.  
**Mitigations:** R1.1 (all endpoints require auth), R2.1 (localhost default), R2.2 (TLS required for remote).  
**Residual risk:** Very low — no auth bypass possible by design.

### T3 — Cross-Tenant Data Access
**Attack:** Compromised user or bug causes one tenant's data to be accessible to another.  
**Mitigations:** R1.6 (`tenant_id` validated at every layer), R5.4 (all queries scoped by tenant), R3.3 (filesystem scoped to workspace).  
**Residual risk:** Low — requires simultaneous failure at multiple independent layers.

### T4 — Malicious Skill Installation
**Attack:** Attacker convinces user to install a malicious skill containing backdoor code.  
**Mitigations:** R6.3 (GPG-signed manifests, local-only, human approval), R3.2 (tool sandbox isolation), R8.2 (CI skill linter).  
**Residual risk:** Medium — human approval step is a control; social engineering possible. Mitigated by sandbox isolation.

### T5 — Tool Filesystem Escape
**Attack:** Malicious tool input attempts path traversal (`../../etc/passwd`).  
**Mitigations:** R3.3 (path normalisation and workspace root validation), R3.2 (container filesystem isolation).  
**Residual risk:** Very low — two independent layers: application-level normalisation + container bind-mount scope.

### T6 — LLM API Key Exfiltration
**Attack:** Prompt injection causes agent to reveal LLM API key in response.  
**Mitigations:** R4.2 (SecretRef — key never in memory as plaintext string beyond resolution point), R4.5 (audit logger redaction), output guard, R6.2 canary tokens.  
**Residual risk:** Low.

### T7 — Supply Chain Compromise
**Attack:** Malicious package injected into dependencies or container image.  
**Mitigations:** R8.2 (Snyk SCA, Trivy), R8.4 (pinned deps), R8.1 (signed images).  
**Residual risk:** Low for known CVEs; medium for zero-day supply chain attacks (industry-wide risk).

### T8 — Audit Log Tampering
**Attack:** Attacker with partial system access modifies audit logs to cover tracks.  
**Mitigations:** R7.2 (Ed25519 signatures on every entry), R7.3 (cold storage with object lock), Vault audit log as secondary evidence.  
**Residual risk:** Very low — forging requires private signing key (in Vault, MFA-protected).

---

## 4. Out of Scope

- Physical access to host hardware
- Compromise of cloud provider infrastructure
- Zero-day vulnerabilities in Linux kernel or Docker daemon
- Social engineering of user outside the system

These are mitigated at the deployment level (hardened host, reputable cloud provider) but are not in-scope for SecureClaw's own controls.
