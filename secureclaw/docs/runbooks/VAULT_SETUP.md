# Vault Setup Runbook — SecureClaw

**Version:** 1.0  
**Date:** March 2026  

This runbook covers first-time Vault initialisation for SecureClaw Phase 1 (homelab, Transit auto-unseal).

---

## Prerequisites

- Docker and Docker Compose installed.
- Vaultwarden or a password manager available to store unseal keys.
- SecureClaw `secureclaw` branch cloned locally.

---

## Step 1: Start Transit Vault (first time only)

```bash
cd secureclaw/deploy

# Start only the transit vault service first
docker compose up -d vault-transit

# Wait for healthy
docker compose exec vault-transit vault status
```

---

## Step 2: Initialise Transit Vault

```bash
# Initialise with 1-of-1 key shares (homelab simplicity; increase for team use)
docker compose exec vault-transit vault operator init \
  -key-shares=1 \
  -key-threshold=1 \
  -format=json > /tmp/vault-transit-init.json

# IMMEDIATELY: save the unseal key and root token to your password manager
cat /tmp/vault-transit-init.json
# Store: unseal_keys_b64[0], root_token

# Delete the init file
rm /tmp/vault-transit-init.json

# Unseal transit vault
docker compose exec vault-transit vault operator unseal <unseal_key_from_password_manager>
```

---

## Step 3: Configure Transit Vault for Auto-Unseal

```bash
# Login to transit vault
export VAULT_ADDR=http://127.0.0.1:8210
vault login <root_token_from_password_manager>

# Enable transit secrets engine
vault secrets enable transit

# Create the auto-unseal key for primary vault
vault write -f transit/keys/autounseal

# Create a policy for primary vault to use transit
vault policy write autounseal - <<EOF
path "transit/encrypt/autounseal" { capabilities = ["update"] }
path "transit/decrypt/autounseal" { capabilities = ["update"] }
EOF

# Create a token for primary vault (periodic, renewable)
vault token create \
  -policy=autounseal \
  -period=24h \
  -orphan \
  -format=json | jq -r '.auth.client_token' > secureclaw/deploy/secrets/vault_transit_token.txt

# Lock down the token file
chmod 600 secureclaw/deploy/secrets/vault_transit_token.txt

# IMPORTANT: Add vault_transit_token.txt to .gitignore
echo 'secureclaw/deploy/secrets/' >> .gitignore
```

---

## Step 4: Initialise Primary Vault

```bash
# Start primary vault (it will auto-unseal via transit)
docker compose up -d vault

# Wait for healthy
docker compose exec vault vault status
# Should show: Sealed: false

# Initialise primary vault
docker compose exec vault vault operator init \
  -recovery-shares=5 \
  -recovery-threshold=3 \
  -format=json > /tmp/vault-primary-init.json

# IMMEDIATELY: save all 5 recovery keys and root token to your password manager
cat /tmp/vault-primary-init.json
rm /tmp/vault-primary-init.json
```

---

## Step 5: Seed SecureClaw Secrets

```bash
export VAULT_ADDR=http://127.0.0.1:8200
vault login <primary_root_token>

# Enable KV v2
vault secrets enable -path=secret kv-v2

# Enable Transit for audit signing
vault secrets enable transit
vault write -f transit/keys/audit-signing type=ed25519

# Seed secrets (replace placeholder values with real values)
vault kv put secret/secureclaw/llm \
  anthropic_api_key="sk-ant-REPLACE_ME" \
  openai_api_key="sk-REPLACE_ME"

vault kv put secret/secureclaw/canary \
  secret="$(openssl rand -hex 32)"

vault kv put secret/secureclaw/session \
  signing_secret="$(openssl rand -hex 64)"

vault kv put secret/secureclaw/qdrant \
  read_write_key="$(openssl rand -hex 32)" \
  read_only_key="$(openssl rand -hex 32)"

# Export audit signing public key for verification store
vault read -format=json transit/keys/audit-signing \
  | jq -r '.data.keys["1"].public_key' > secureclaw/deploy/audit-pubkey.pem
```

---

## Step 6: Create Service Policies and Tokens

```bash
# Gateway policy
vault policy write secureclaw-gateway - <<EOF
path "secret/data/secureclaw/session" { capabilities = ["read"] }
path "secret/data/secureclaw/llm" { capabilities = ["read"] }
path "transit/sign/audit-signing" { capabilities = ["update"] }
EOF

# Agent policy
vault policy write secureclaw-agent - <<EOF
path "secret/data/secureclaw/llm" { capabilities = ["read"] }
path "secret/data/secureclaw/canary" { capabilities = ["read"] }
path "secret/data/secureclaw/qdrant" { capabilities = ["read"] }
EOF

# Generate service tokens (written to secrets/ folder for Docker secrets mount)
vault token create -policy=secureclaw-gateway -period=24h -orphan -format=json \
  | jq -r '.auth.client_token' > secureclaw/deploy/secrets/vault_token_gateway.txt

vault token create -policy=secureclaw-agent -period=24h -orphan -format=json \
  | jq -r '.auth.client_token' > secureclaw/deploy/secrets/vault_token_agent.txt

chmod 600 secureclaw/deploy/secrets/vault_token_gateway.txt
chmod 600 secureclaw/deploy/secrets/vault_token_agent.txt
```

---

## Step 7: Start Full Stack

```bash
docker compose up -d

# Verify all services healthy
docker compose ps

# Check vault is unsealed
docker compose exec vault vault status | grep Sealed
# Expected: Sealed    false
```

---

## Emergency Procedures

### Seal Primary Vault Immediately
```bash
docker compose exec vault vault operator seal
# Or: stop vault-transit to prevent future auto-unseal
docker compose stop vault-transit
```

### Rotate All Service Tokens
```bash
# Run first-run.sh with --rotate-tokens flag (to be implemented in Stage 1.2)
bash secureclaw/deploy/first-run.sh --rotate-tokens
```

---

## Notes

- `secureclaw/deploy/secrets/` is gitignored — never commit token files.
- Transit Vault unseal key and root token must be in your password manager only.
- Vault audit log is at `vault-logs/vault_audit.log` in the named Docker volume.
- For team/enterprise use, replace 1-of-1 transit key shares with 3-of-5.
