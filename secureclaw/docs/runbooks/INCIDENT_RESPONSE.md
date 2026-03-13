# SecureClaw Incident Response Runbook

**Version:** 1.0  
**Date:** March 2026  

---

## Severity Levels

| Level | Criteria | Response Time | Auto-Action |
|---|---|---|---|
| P1 Critical | Canary trigger, credential exfil suspected, RCE indicator | Immediate | Session quarantine, gateway alert |
| P2 High | Repeated policy denials (>10 in 5min), auth anomaly | <15 min | User session suspended, alert |
| P3 Medium | Single policy denial, config lint failure | <1 hour | Log + alert |
| P4 Low | Informational anomaly, rate limit hit | <24 hours | Log only |

---

## Runbook: P1 — Canary Token Triggered

1. System automatically quarantines the affected session (agent suspended, no further tool calls).
2. Audit log snapshot exported and integrity-verified (check Ed25519 signatures).
3. Affected tenant workspace placed in read-only mode pending investigation.
4. Rotate all secrets associated with affected tenant (Vault: `vault lease revoke -prefix secret/<tenant>`).
5. Review audit log for: which tool calls were made, what data was accessed, any exfil indicators.
6. Determine whether prompt injection was the vector or a system bug.
7. Remediate and re-enable tenant after root cause confirmed.

---

## Runbook: P2 — Repeated Policy Denials

1. Review policy denial audit events: which user, which tool, which arguments.
2. Determine if legitimate (misconfigured policy) or attack (probing).
3. If attack: suspend user session, notify admin via configured webhook.
4. If misconfigured: adjust policy and document change in ADR.

---

## Runbook: Rotate All Secrets

```bash
# Rotate LLM API keys
vault kv put secret/secureclaw/llm/anthropic api_key=<new_key>
vault kv put secret/secureclaw/llm/openai api_key=<new_key>

# Revoke all existing service tokens and re-issue
vault token revoke -accessor <accessor_id>

# Rotate audit signing key
vault write transit/keys/audit-signing/rotate

# Restart services to pick up new tokens
docker compose restart gateway agent
```

---

## Runbook: Kill Gateway (Emergency Stop)

```bash
docker compose stop gateway
# Verify all external connections terminated
docker compose logs gateway | tail -20
# Audit log will record the shutdown event automatically
```

---

## Contact & Escalation

Configure `SECURECLAW_ALERT_WEBHOOK` in environment to receive automated incident alerts.
For open-source deployments, raise a private security issue at the repo security advisory page.
