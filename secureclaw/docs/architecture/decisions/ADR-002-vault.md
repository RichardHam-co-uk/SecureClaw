# ADR-002: HashiCorp Vault as Secrets Backend

**Status:** Accepted  
**Date:** March 2026  

## Context

OpenClaw stores secrets as plaintext files in `~/.openclaw/credentials/`. This is a critical security deficiency we must not replicate.

## Decision

Deploy HashiCorp Vault as a containerised service in the Docker Compose stack. All secrets are accessed via short-lived Vault tokens. Human access requires TOTP MFA (FIDO2 for privileged paths). Secrets are referenced in config via `SecretRef` descriptors, never resolved until runtime.

## Rationale

- Vault is battle-tested, audited, and widely trusted in enterprise environments.
- Containerised deployment keeps it self-contained and air-gap compatible.
- Short-lived tokens limit blast radius of token compromise.
- MFA gate aligns with R4.4 of the SRS.
- Vault audit log provides additional evidence trail independent of SecureClaw's own audit system.

## Consequences

- Vault must be unsealed before SecureClaw can start — adds operational step.
- Need to document Vault initialisation and unseal process.
- Consider auto-unseal (cloud KMS or local HSM) for production environments.
