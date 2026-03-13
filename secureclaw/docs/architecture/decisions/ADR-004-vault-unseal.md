# ADR-004: Vault Unseal Strategy

**Status:** Accepted  
**Date:** March 2026  

---

## Context

HashiCorp Vault requires unsealing after every restart. Manual unseal (entering 3 of 5 Shamir key shares at the CLI) is secure but operationally painful for a homelab environment where Docker restarts frequently. We need an automated unseal strategy that does not compromise the security model and scales from homelab through to enterprise.

## Options Considered

### Option A — Manual Unseal
- Operator enters key shares manually after every restart.
- Maximum security, maximum operational friction.
- Unsuitable for homelab / automated CI environments.

### Option B — Vault Transit Auto-Unseal (local second Vault instance)
- A second minimal "transit" Vault instance (HA pair) holds the unseal key.
- Primary Vault calls Transit Vault at boot to decrypt its unseal key.
- Both instances can be local; no cloud dependency.
- If Transit Vault is down, primary Vault cannot unseal — provides a kill switch.

### Option C — Cloud KMS Auto-Unseal (AWS KMS / GCP CKMS / Azure Key Vault)
- Primary Vault calls cloud KMS at boot for unseal key decryption.
- Excellent for Phase 2 enterprise deployments.
- Creates cloud dependency — breaks air-gap requirement (SRS R8.5).
- Not suitable for Phase 1 homelab.

### Option D — Local Keyfile Script
- Encrypted keyfile stored on host; a startup script decrypts and passes keys to Vault.
- Simple to implement but keyfile protection relies entirely on host filesystem security.
- Lowest security of all options.

## Decision

**Phase 1 (homelab):** Option B — Vault Transit Auto-Unseal with a local secondary Vault container in the Docker Compose stack.

**Phase 2 (enterprise):** Option C — Cloud KMS. The Vault config abstraction means switching requires only a config change, not code change.

## Implementation (Phase 1)

```
docker-compose services:
  vault-transit:   # Secondary Vault — Transit secrets engine only
    - Binds to backend network only
    - Initialised with single unseal key (stored in Vault operator's password manager)
    - Exposes Transit secrets engine on path: transit/
    - MFA required for human access (TOTP)

  vault:           # Primary Vault — all SecureClaw secrets
    - Configured to use Transit auto-unseal pointing at vault-transit
    - Starts unsealed automatically as long as vault-transit is healthy
    - If vault-transit is stopped, vault seals itself on next restart (kill switch)
```

## Unseal Key Protection

- Transit Vault root token and unseal keys generated once at `first-run.sh`.
- Stored in operator's password manager (Vaultwarden recommended — aligns with existing homelab).
- Never stored on disk in plaintext.
- TOTP MFA required for any human access to vault-transit.

## Vault Initialisation Runbook

See `docs/runbooks/VAULT_SETUP.md` (to be created in Stage 1.2).

## Consequences

- Docker Compose stack gains a second Vault container (`vault-transit`).
- Slightly more complex first-run initialisation.
- Provides automatic unseal on restart without cloud dependency.
- Kill switch: stopping `vault-transit` prevents primary Vault from unsealing.
- Phase 2 migration to cloud KMS requires only Vault config change.
