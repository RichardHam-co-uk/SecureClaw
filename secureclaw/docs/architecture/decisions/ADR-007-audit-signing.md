# ADR-007: Ed25519 Audit Log Signing — Key Management

**Status:** Accepted  
**Date:** March 2026  

---

## Context

SRS R7.2 requires Ed25519 signatures on every audit log entry for tamper-evidence. We need to decide how the signing key is stored and accessed without creating a circular dependency (the audit logger needs to sign events, but it accesses Vault for the key, and Vault access itself generates audit events).

## Decision

Use **Vault Transit Secrets Engine** for Ed25519 signing.

- The audit signing key is created as a Vault Transit key of type `ed25519`.
- The Audit Logger calls Vault Transit `sign` endpoint with each event's payload hash.
- Vault Transit performs the signing inside Vault — the private key **never leaves Vault**.
- The public key is exported once at first-run and stored in the audit database for verification.

## Circular Dependency Resolution

Vault's own audit log and SecureClaw's audit log are separate concerns:
- Vault audit log: records Vault API access (enabled in Vault config, writes to file).
- SecureClaw audit log: records agent/tool/policy events (signed via Vault Transit).
- The SecureClaw audit logger does **not** emit an audit event for its own Vault signing calls — this would cause infinite recursion. Vault signing calls are captured by Vault's own audit log instead.

## Key Rotation

- Transit keys support versioned rotation: `vault write -f transit/keys/audit-signing/rotate`.
- Old key versions retained for verification of historical entries.
- Rotation triggered: annually on schedule, or immediately on suspected compromise.
- Rotation recorded as a special audit event (signed with the new key).

## Verification

```bash
# Export current public key
vault read transit/keys/audit-signing

# Verify a log entry (pseudocode)
pubkey = export_from_vault()
signature = base64_decode(log_entry.signature)
payload_hash = sha256(log_entry.payload_canonical_json)
ed25519_verify(pubkey, payload_hash, signature)  # must return True
```

## Consequences

- Private key never exists outside Vault — strongest possible key protection.
- Signing adds ~5ms latency per audit event (Vault Transit API call).
- Vault must be unsealed for audit logging to function — this is intentional (if Vault is down, we know immediately).
- Public key export needed once at first-run for verification tooling.
