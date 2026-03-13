# 🔒 SecureClaw — Hardened AI Assistant Gateway

<p align="center">
  <strong>OpenClaw, security-hardened with HashiCorp Vault, secret-zero elimination, and production-grade secret management.</strong>
</p>

<p align="center">
  <a href="https://github.com/RichardHam-Co-Uk/secureclaw/actions/workflows/ci.yml?branch=main"><img src="https://img.shields.io/github/actions/workflow/status/RichardHam-Co-Uk/secureclaw/ci.yml?branch=main&style=for-the-badge" alt="CI status"></a>
  <a href="https://github.com/RichardHam-Co-Uk/secureclaw/releases"><img src="https://img.shields.io/github/v/release/RichardHam-Co-Uk/secureclaw?include_prereleases&style=for-the-badge" alt="GitHub release"></a>
  <a href="LICENSE"><img src="https://img.shields.io/badge/License-MIT-blue.svg?style=for-the-badge" alt="MIT License"></a>
</p>

> **SecureClaw** is a security-hardened fork of [OpenClaw](https://github.com/openclaw/openclaw) — a personal AI assistant gateway you run on your own infrastructure.  
> This fork extends the upstream project with enterprise-grade secret management, Vault integration, and a hardened deployment architecture designed for security-conscious operators.

---

## What SecureClaw Adds

The upstream OpenClaw project is excellent. SecureClaw layers the following on top:

### 🔐 HashiCorp Vault Integration
- **KV v2** secrets backend for all application secrets
- **Transit secrets engine** for encryption-as-a-service and auto-unseal
- `VaultClient` with auto token renewal, retry backoff, and graceful degradation
- Vault agent sidecar pattern for secret injection

### 🚫 Secret-Zero Elimination
- `SecretRef` URI pattern: `vault://path/to/secret#field` and `env://VAR_NAME`
- No secrets ever written to disk, environment files, or container logs
- All secret resolution happens at runtime via the `SecretRef` resolver

### 🐳 Hardened Docker Compose
- Dedicated `vault-transit` service with Transit auto-unseal wired up
- `qdrant` vector store service for semantic memory
- Immutable container filesystem where possible
- Non-root service execution throughout

### 🛠️ Operator-Grade First-Run Script
- `first-run.sh` — idempotent 15-step initialisation script
- Colour output, `--rotate-tokens` flag
- Never prints secrets to stdout
- Full Vault initialisation, unsealing, policy and AppRole setup

---

## Repository Structure

```
openclaw/               ← Upstream OpenClaw source (synced from openclaw/openclaw)
secureclaw/             ← SecureClaw hardening layer
  ├── docker/           ← Hardened compose + Vault config
  ├── src/              ← VaultClient, SecretRef resolver, extensions
  ├── scripts/          ← first-run.sh and operator tooling
  └── docs/
      ├── architecture/ ← System design and ADRs
      ├── development/  ← Claude Code prompts and dev guides
      ├── runbooks/     ← Operational runbooks
      └── security/     ← Threat model, security controls
```

---

## Branch Strategy

| Branch | Purpose |
|---|---|
| `main` | Fork baseline; periodically synced from upstream |
| `secureclaw` | Active SecureClaw development — all hardening work lives here |
| `release/v*` | Tagged stable SecureClaw releases |

See [FORK_NOTES.md](FORK_NOTES.md) for upstream sync guidance.

---

## Getting Started

SecureClaw is under active development. The hardening layer is being built in stages:

- **Stage 1 (In Progress):** Vault integration, SecretRef pattern, hardened compose, first-run script
- **Stage 2 (Planned):** Prompt injection hardening, agent sandboxing, audit logging
- **Stage 3 (Planned):** mTLS between services, secrets rotation automation, compliance controls

For the underlying OpenClaw functionality (channels, agents, voice, canvas), refer to the [upstream documentation](https://docs.openclaw.ai).

### Prerequisites

- Docker + Docker Compose v2
- HashiCorp Vault (provided via compose or external)
- Node ≥ 22 (for upstream OpenClaw runtime)

### Quick Start (once Stage 1 is complete)

```bash
git clone https://github.com/RichardHam-Co-Uk/secureclaw.git
cd secureclaw
git checkout secureclaw

# Initialise Vault and all services
bash secureclaw/scripts/first-run.sh

# Start the hardened stack
docker compose -f secureclaw/docker/docker-compose.yml up -d
```

---

## Security Model

SecureClaw is built on a **defence-in-depth** model:

- All secrets resolved at runtime via `SecretRef` — no plaintext secrets in config files or environment
- Vault Transit provides encryption-as-a-service; the application never holds raw encryption keys
- Container images run non-root; filesystems are immutable where the runtime permits
- Prompt injection risk mitigated via agent sandboxing (Stage 2)
- Full audit trail via Vault audit log + structured application logging

---

## Development

Active development uses Claude Code with structured prompts. See:

- [`secureclaw/docs/development/CLAUDE_CODE_PROMPTS.md`](secureclaw/docs/development/CLAUDE_CODE_PROMPTS.md) — Session prompts for each build milestone
- [`secureclaw/docs/architecture/`](secureclaw/docs/architecture/) — Architecture decision records
- [`secureclaw/docs/security/`](secureclaw/docs/security/) — Threat model and security controls

---

## Attribution

SecureClaw is a fork of **[OpenClaw](https://github.com/openclaw/openclaw)** by [Peter Steinberger](https://steipete.me) and the OpenClaw community, licensed under the [MIT License](LICENSE).

All upstream functionality and intellectual property remains the work of the OpenClaw contributors. This fork adds security hardening on top of that foundation.

---

## Contributing

This is currently a personal/private hardening project. Contributions welcome once the architecture stabilises post-Stage 1.
