# SecureClaw — Dependency Register

**Version:** 1.0  
**Date:** March 2026  
**Purpose:** Pin all runtime dependencies to minimum safe versions. Updated after every security research review and CVE assessment. All versions confirmed CVE-free at date of entry.

> **Rule:** No dependency may be added or upgraded without:
> 1. Checking Snyk (`snyk test`) and pip-audit for CVEs.
> 2. Adding/updating the entry in this document.
> 3. Committing the lockfile change alongside this document update.

---

## Runtime Infrastructure

| Component | Image / Package | Minimum Safe Version | Pinned Version | CVE Check Date | Notes |
|---|---|---|---|---|---|
| Node.js | `node` | `20.19.0 LTS` | `20.19.0-alpine` | March 2026 | LTS stream only |
| Python | `python` | `3.12.9` | `3.12.9-slim` | March 2026 | Slim base image |
| HashiCorp Vault | `hashicorp/vault` | `1.18.0` | `1.18.3` | March 2026 | Pin to patch version |
| Qdrant | `qdrant/qdrant` | `1.13.0` | `1.13.2` | March 2026 | Collection-per-tenant |
| Grafana Loki | `grafana/loki` | `3.4.0` | `3.4.2` | March 2026 | |
| Prometheus | `prom/prometheus` | `3.2.0` | `3.2.1` | March 2026 | |
| Grafana | `grafana/grafana` | `11.6.0` | `11.6.1` | March 2026 | |

---

## TypeScript / Node.js (Gateway, Policy Engine, Audit Logger, Vault Adapter)

| Package | Purpose | Minimum Safe Version | CVE Check Date | Notes |
|---|---|---|---|---|
| `jose` | JWT validation (OIDC/local token) | `^5.9.6` | March 2026 | Only approved JWT library. No custom parsing. |
| `zod` | Schema validation | `^3.24.1` | March 2026 | All inputs validated before use |
| `hono` | HTTP framework (gateway) | `^4.7.4` | March 2026 | Lightweight, TypeScript-first, no prototype pollution CVEs |
| `@hono/node-server` | Node.js adapter for Hono | `^1.13.7` | March 2026 | |
| `ws` | WebSocket server | `^8.18.1` | March 2026 | Patched for DoS CVE in <8.17 |
| `pino` | Structured logging | `^9.6.0` | March 2026 | JSON output for Loki ingestion |
| `winston` | Fallback logger | Not used | — | Excluded: heavier, fewer security guarantees than pino |
| `@node-rs/ed25519` | Ed25519 signing (audit log) | `^1.7.0` | March 2026 | Native bindings, faster than pure JS |
| `node-vault` | HashiCorp Vault client | `^0.10.0` | March 2026 | Actively maintained |
| `rate-limiter-flexible` | Rate limiting | `^5.0.4` | March 2026 | Sliding window, Redis-compatible for Phase 2 |
| `vitest` | Test framework | `^3.0.9` | March 2026 | Dev only |
| `typescript` | TypeScript compiler | `^5.8.2` | March 2026 | Dev only |
| `tsdown` | Build tool | `^0.9.4` | March 2026 | Dev only |

**Excluded packages (and why):**
| Package | Reason for Exclusion |
|---|---|
| `jsonwebtoken` | Multiple historical CVEs; replaced by `jose` |
| `express` | Heavier surface area than Hono; prototype pollution history |
| `axios` | SSRF history; use native `fetch` or controlled HTTP client |
| `lodash` | Prototype pollution CVEs; use native JS |
| `moment` | Unmaintained; use `date-fns` or `Temporal` |

---

## Python (Agent Runtime, Tools, Memory)

| Package | Purpose | Minimum Safe Version | CVE Check Date | Notes |
|---|---|---|---|---|
| `litellm` | LLM abstraction layer (multi-provider) | `>=1.82.1` | March 2026 | **Critical:** <1.82.0 has unauthenticated RCE in custom code guardrail. Never enable custom_code_guardrail feature. |
| `pydantic` | Input validation | `>=2.11.1` | March 2026 | V2 only — V1 has known ReDoS issues |
| `qdrant-client` | Qdrant vector DB client | `>=1.13.2` | March 2026 | Matches server version pin |
| `cryptography` | Ed25519, encryption at rest | `>=44.0.2` | March 2026 | OpenSSL bindings; pin to patch |
| `hvac` | HashiCorp Vault Python client | `>=2.3.0` | March 2026 | |
| `structlog` | Structured logging | `>=25.1.0` | March 2026 | JSON output for Loki |
| `httpx` | Controlled HTTP client (tools) | `>=0.28.1` | March 2026 | Used only in allowlisted tool contexts |
| `pip-audit` | Dep vulnerability scanner | `>=2.8.0` | March 2026 | Dev / CI tool |
| `pytest` | Test framework | `>=8.3.4` | March 2026 | Dev only |
| `pytest-asyncio` | Async test support | `>=0.25.2` | March 2026 | Dev only |
| `uv` | Package manager | `>=0.6.6` | March 2026 | Faster than pip, reproducible installs |

**Excluded packages (and why):**
| Package | Reason for Exclusion |
|---|---|
| `chromadb` | No native auth or DB-level tenant isolation (see ADR-005) |
| `langchain` | Large attack surface; indirect deps with recurring CVEs; LiteLLM preferred |
| `openai` (direct) | Use via LiteLLM abstraction only; never direct |
| `requests` | Use `httpx` instead — async, better timeout handling, SSRF mitigations |
| `pickle` | Arbitrary code execution on deserialise; never use for data persistence |

---

## CI / Security Tooling

| Tool | Purpose | Version / Channel | Notes |
|---|---|---|---|
| Semgrep | SAST | `latest` (CI) | Rules: `auto`, `p/typescript`, `p/python`, `p/owasp-top-ten` |
| Snyk | SCA (Node + Python) | `latest` (CI) | Requires `SNYK_TOKEN` secret in GitHub Actions |
| Trivy | Container image scanning | `latest` (CI) | Fail on CRITICAL/HIGH |
| pip-audit | Python dep CVE scan | `>=2.8.0` | Run in CI and pre-commit |
| pnpm audit | Node dep CVE scan | Built-in to pnpm | Run in Dockerfile build stage |

---

## Dependency Update Policy

1. **Security patches** (CVE fix): Update within 48 hours of disclosure. Immediate PR, fast-track review.
2. **Minor updates**: Monthly review cycle. Update if no breaking changes and no new CVEs.
3. **Major updates**: Planned, with full regression test run and Codex security review before merge.
4. **Automated PRs**: Dependabot/Renovate enabled on `secureclaw` branch. All automated PRs go through full CI security gates before merge.

---

## CVE Watch List

Known historical CVEs relevant to our stack — monitored for recurrence:

| Package | CVE | Severity | Status | Notes |
|---|---|---|---|---|
| `litellm` | Custom code guardrail RCE | Critical | Patched in 1.82.0 | Feature disabled in our config |
| `litellm` | Multiple in <1.80.5 | High | Patched | Pin to >=1.82.1 |
| `ws` | DoS via header abuse | High | Patched in 8.17 | Pinned to >=8.18.1 |
| `jsonwebtoken` | Algorithm confusion | Critical | N/A | Not used — jose only |
| `chromadb` | No auth by design | Critical | By design | Not used — see ADR-005 |
