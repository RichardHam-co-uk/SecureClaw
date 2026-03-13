# Claude Code Prompts — SecureClaw

Use these prompt templates when starting each Claude Code session. Always include the governing documents.

---

## Session Opener (use every time)

```
I am building SecureClaw, a secure-by-design AI agent platform.

Before writing any code, read and internalize these governing documents:
1. secureclaw/CLAUDE.md — project overview and security mandates
2. secureclaw/docs/security/SRS.md — security requirements specification
3. secureclaw/docs/architecture/ARCHITECTURE.md — system architecture

All code you generate MUST comply with the SRS. If any requirement conflicts
with a feature I ask for, flag the conflict before writing code.
```

---

## Gateway Module

```
Implement the SecureClaw Gateway auth module (secureclaw/gateway/src/auth/).

Requirements from SRS:
- R1.1: All endpoints require auth. No unauthenticated access.
- R1.2: Support OIDC/JWT (using 'jose' library only) and local signed tokens.
- R1.4: Session tokens scoped to tenant_id + user_id, expire ≤24h.
- R2.4: Rate limiting on all endpoints.

Generate:
1. TypeScript interface for AuthProvider (supports OIDC and local token)
2. JWT validation middleware using 'jose' (not custom parsing)
3. Session token type with tenant_id, user_id, role, expiry
4. Rate limiter middleware (sliding window, per-IP and per-user)
5. Unit tests for all auth paths including failure cases

Do not generate any auth bypass, default credentials, or 'skip auth in dev' patterns.
```

---

## Policy Engine Module

```
Implement the SecureClaw Policy Engine (secureclaw/policy/src/).

Requirements from SRS:
- R5.1: Central decide() function for all tool calls
- R5.2: Secure defaults — exec, browser, write_file, http_client disabled by default
- R5.3: Per-tenant allow/block lists
- R5.5: All decisions logged as audit events

Generate:
1. PolicyDecision type: { allow: boolean, reason: string, tenant_id: string, user_id: string, tool_name: string, timestamp: Date }
2. decide() function with RBAC evaluation
3. Default policy config (all dangerous tools disabled)
4. Per-tenant policy override loading from config
5. Integration with AuditLogger interface
6. Unit tests covering: allow, deny, missing role, disabled tool, override cases
```

---

## Prompt Injection Detection

```
Implement the SecureClaw prompt injection detection system.

Requirements from SRS R6.2:
- Layer 1: Static pre-pass regex + semantic patterns
- Layer 3: Canary token injection and monitoring
- Layer 4: Output guard before tool call construction

Generate:
1. Python class InjectionDetector with detect(input: str) -> DetectionResult
2. Pattern library covering: role override attempts, ignore/disregard instructions,
   system prompt exfil, tool argument injection, jailbreak patterns
3. CanaryManager: inject_canary(session_id) -> str, check_output(output: str, session_id) -> bool
4. OutputGuard: validate_tool_call(llm_output: dict) -> ValidatedToolCall | GuardViolation
5. Unit tests with 20+ adversarial examples that must all be detected
```

---

## Audit Logger Module

```
Implement the SecureClaw Audit Logger (secureclaw/audit/src/).

Requirements from SRS:
- R7.1: Structured events for all audit points
- R7.2: Ed25519 signatures on every entry
- R7.3: Loki sink (hot), local file (warm)
- R4.5: Secret redaction before signing

Generate:
1. AuditEvent schema (TypeScript Zod schema)
2. SecretRedactor: redact(payload: object) -> object (regex-based, configurable patterns)
3. AuditLogger class: log(event: AuditEvent) -> void (redact, sign, emit)
4. Ed25519 signer using key from Vault via SecretRef
5. Loki transport and local file transport
6. Unit tests verifying: redaction works, signature verifiable, tamper detection
```
