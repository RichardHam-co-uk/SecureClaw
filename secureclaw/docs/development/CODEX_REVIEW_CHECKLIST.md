# Codex Security Review Checklist — SecureClaw

Use this checklist when running Codex review on every module and PR.
Paste the module code + this checklist into Codex and ask it to evaluate each item.

---

## Authentication & Authorisation
- [ ] Are all endpoints protected by authentication middleware?
- [ ] Is JWT/token validation using a well-audited library (no custom parsing)?
- [ ] Is `tenant_id` validated on every request at every layer?
- [ ] Are session tokens properly scoped and expiring?
- [ ] Is step-up MFA enforced for admin/privileged operations?

## Input Validation
- [ ] Are all inputs validated against a schema (Zod/Pydantic) before use?
- [ ] Is unvalidated data prevented from reaching business logic?
- [ ] Are all file paths normalised and checked against workspace root?
- [ ] Are prompt injection patterns checked before LLM calls?

## Secrets & Credentials
- [ ] Are there any hardcoded secrets, tokens, or passwords?
- [ ] Are all secret references using `SecretRef` descriptors?
- [ ] Are secrets redacted from all log/audit outputs?
- [ ] Are secrets zeroed from memory after use?

## Tool Execution
- [ ] Do all tool calls go through the Policy Engine before execution?
- [ ] Is sandboxing enforced (no direct subprocess/shell calls outside sandbox)?
- [ ] Are path traversal attempts (`../`, absolute paths) prevented?
- [ ] Is network egress limited to allowlisted destinations?

## Audit & Logging
- [ ] Are all security-relevant events emitted via AuditLogger?
- [ ] Is `console.log` or equivalent used for security events? (should not be)
- [ ] Are audit entries signed before persistence?
- [ ] Are secrets redacted before audit events are signed?

## Dependencies
- [ ] Are all new dependencies pinned to exact versions?
- [ ] Have new dependencies been checked for CVEs?
- [ ] Are there any unnecessary dependencies that increase attack surface?

## Error Handling
- [ ] Are errors handled gracefully without leaking stack traces or internal state to users?
- [ ] Are authentication failures returning consistent error messages (no oracle)?
- [ ] Are failed policy decisions logged before the error is returned?

## Tests
- [ ] Are security-critical paths covered by unit tests?
- [ ] Are failure/rejection paths tested as well as happy paths?
- [ ] Are adversarial inputs included in test cases?
