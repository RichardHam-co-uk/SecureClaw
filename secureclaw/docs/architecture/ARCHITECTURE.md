# SecureClaw Architecture

**Version:** 1.0  
**Date:** March 2026  

---

## Overview

SecureClaw is structured as a multi-service system with strong isolation between components. All communication between services passes through defined, typed interfaces — never direct function calls across trust boundaries.

```
┌─────────────────────────────────────────────────────────────┐
│                        EXTERNAL WORLD                       │
│  CLI │ Web UI │ Discord │ Slack │ Teams │ REST API           │
└──────────────────────┬──────────────────────────────────────┘
                       │  TLS / Auth required
                       ▼
┌─────────────────────────────────────────────────────────────┐
│                     GATEWAY (TypeScript)                    │
│  ┌─────────────┐  ┌──────────────┐  ┌───────────────────┐  │
│  │  Auth/OIDC  │  │  Rate Limit  │  │  Schema Validator │  │
│  └─────────────┘  └──────────────┘  └───────────────────┘  │
│  ┌──────────────────────────────────────────────────────┐   │
│  │              Channel Adapters                        │   │
│  │  CLI  │  WebSocket UI  │  Discord  │  Slack  │ Teams │   │
│  └──────────────────────────────────────────────────────┘   │
└──────────────────────┬──────────────────────────────────────┘
                       │  Validated, authenticated request
                       ▼
┌─────────────────────────────────────────────────────────────┐
│                  POLICY ENGINE (TypeScript)                 │
│  decide(user, tenant, role, tool, args) → allow/deny + log  │
│  ┌─────────────┐  ┌──────────────┐  ┌───────────────────┐  │
│  │  RBAC Rules │  │  Tool Policy │  │  Prompt Guard     │  │
│  └─────────────┘  └──────────────┘  └───────────────────┘  │
└──────────────────────┬──────────────────────────────────────┘
                       │  Approved request
                       ▼
┌─────────────────────────────────────────────────────────────┐
│                AGENT RUNTIME (Python)                       │
│  ┌─────────────────────────────────────────────────────┐   │
│  │  Context Assembler                                  │   │
│  │  (system prompt + memory + workspace + tenant ctx)  │   │
│  └───────────────────────┬─────────────────────────────┘   │
│                          │                                  │
│  ┌───────────────────────▼─────────────────────────────┐   │
│  │  LLM Abstraction Layer (LiteLLM)                    │   │
│  │  Anthropic │ OpenAI │ Ollama │ Gemini │ ...         │   │
│  └───────────────────────┬─────────────────────────────┘   │
│                          │  Tool call request              │
│  ┌───────────────────────▼─────────────────────────────┐   │
│  │  Output Guard (validate LLM output before exec)     │   │
│  └───────────────────────┬─────────────────────────────┘   │
└──────────────────────────┼──────────────────────────────────┘
                           │  → Policy Engine (re-check)
                           ▼
┌─────────────────────────────────────────────────────────────┐
│                  TOOL SANDBOX (Docker/Podman)               │
│  Per-invocation isolated container                          │
│  ┌────────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐  │
│  │ read_file  │  │web_search│  │struct_qry│  │[opt-in]  │  │
│  │ (ws-scoped)│  │(allowlist│  │(read-only│  │exec/http │  │
│  └────────────┘  └──────────┘  └──────────┘  └──────────┘  │
└──────────────────────────────────────────────────────────────┘
           │                              │
           ▼                              ▼
┌─────────────────────┐      ┌────────────────────────────────┐
│  VAULT (HashiCorp)  │      │  MEMORY (Python)               │
│  Secrets backend    │      │  Tenant-scoped vector store     │
│  MFA-protected      │      │  Encrypted at rest             │
└─────────────────────┘      └────────────────────────────────┘
           │
           ▼
┌─────────────────────────────────────────────────────────────┐
│              AUDIT & OBSERVABILITY                          │
│  ┌──────────────┐  ┌──────────┐  ┌────────────────────────┐ │
│  │ Audit Logger │  │ Loki     │  │ Prometheus + Grafana   │ │
│  │ (Ed25519 sig)│  │ (logs)   │  │ (metrics + dashboards) │ │
│  └──────────────┘  └──────────┘  └────────────────────────┘ │
└─────────────────────────────────────────────────────────────┘
```

---

## Service Definitions

### Gateway (TypeScript / Node.js 20)

**Responsibility:** Single entry point for all external connections. Terminates auth, validates schemas, routes to agent.

**Key modules:**
- `auth/`: OIDC/JWT validation (jose), local token validation, session management
- `ratelimit/`: Per-IP and per-user sliding window rate limiter
- `validator/`: Zod schemas for all inbound message types
- `channels/`: CLI, WebSocket UI, Discord, Slack, Teams adapters
- `router/`: Maps validated requests to agent dispatch

**Does NOT:** Execute tools, call LLMs, read files, access secrets directly.

---

### Policy Engine (TypeScript)

**Responsibility:** Evaluate every tool call request before execution. Single `decide()` function; all rules declarative.

**Key modules:**
- `rbac/`: Role definitions and role-to-tool mappings
- `tool-policy/`: Per-tool, per-tenant allow/block rules
- `prompt-guard/`: Static injection pattern detection
- `decision-log/`: Emit audit event for every decision

**Does NOT:** Execute tools, call LLMs, hold state between requests (stateless evaluator).

---

### Agent Runtime (Python 3.12)

**Responsibility:** LLM orchestration loop. Assembles context, calls LLM, parses output, requests tool execution.

**Key modules:**
- `context/`: System prompt assembly, memory retrieval, workspace file injection
- `llm/`: LiteLLM adapter with provider config from SecretRef
- `output_guard/`: Validate and sanitise LLM output before tool construction
- `canary/`: Inject and monitor canary tokens per session
- `loop/`: Main agentic loop with max-iteration and timeout guards

**Does NOT:** Execute tools directly — requests execution via tool sandbox IPC. Does not hold secrets.

---

### Tool Sandbox (Docker/Podman containers)

**Responsibility:** Execute approved tool invocations in isolation.

**Default tools (secure mode):**
- `read_file`: Read files within tenant workspace root only
- `web_search`: Search via allowlisted API (no raw HTTP)
- `structured_query`: Read-only queries against tenant memory

**Opt-in tools (require `operator`/`admin` role + explicit config):**
- `write_file`: Write within workspace, no path traversal
- `http_client`: HTTP requests to allowlisted domains only
- `script_runner`: Execute pre-approved script templates only (no arbitrary shell)
- `browser`: Headless browser, allowlisted domains only

---

### Vault Adapter (TypeScript)

**Responsibility:** Abstract secret resolution. Translate `SecretRef` descriptors to resolved values at runtime.

- `env://VAR` → `process.env.VAR`
- `vault://secret/path` → HashiCorp Vault API call with short-lived token

All resolved values zeroed from memory after use. Never logged.

---

### Audit Logger (TypeScript)

**Responsibility:** Emit structured, signed audit events.

- Event schema: `{id, timestamp, tenant_id, user_id, event_type, payload_hash, signature}`
- Payload redacted of secrets before signing
- Ed25519 key pair: private key in Vault, public key in audit DB for verification
- Emits to: Loki (hot), local file (warm), optional S3 sink (cold)

---

## Data Flow: Secure Request Lifecycle

```
1. User sends message via channel adapter
2. Gateway: authenticate → validate schema → extract tenant/user/role
3. Gateway: forward to Agent Runtime via Unix socket
4. Agent: assemble context (inject canary token)
5. Agent: call LLM with assembled prompt
6. Agent: parse LLM output through output guard
7. Agent: construct tool call request → send to Policy Engine
8. Policy Engine: evaluate decide() → allow/deny
9. If allow: Agent spawns tool sandbox container
10. Tool executes in isolation → returns typed result
11. Agent: validate tool result → continue loop or finalise
12. Gateway: return response to user
13. Audit Logger: sign and emit event for steps 2,4,6,8,10,12
```

---

## Deployment: Docker Compose Services

| Service | Image | Network | Port (internal) |
|---|---|---|---|
| `gateway` | `secureclaw/gateway` | `frontend`, `backend` | 8080 |
| `agent` | `secureclaw/agent` | `backend` | Unix socket |
| `vault` | `hashicorp/vault:latest` | `backend` | 8200 |
| `loki` | `grafana/loki` | `observability` | 3100 |
| `prometheus` | `prom/prometheus` | `observability` | 9090 |
| `grafana` | `grafana/grafana` | `observability` | 3000 |
| `tool-sandbox` | `secureclaw/sandbox` | `sandbox` (isolated) | N/A |

---

## Architecture Decision Records (ADRs)

See `docs/architecture/decisions/` for ADRs covering:
- ADR-001: TypeScript gateway + Python agent pattern
- ADR-002: HashiCorp Vault over embedded secrets manager
- ADR-003: Docker sandbox per tool invocation
- ADR-004: LiteLLM as LLM abstraction layer
- ADR-005: Ed25519 for audit log signing
