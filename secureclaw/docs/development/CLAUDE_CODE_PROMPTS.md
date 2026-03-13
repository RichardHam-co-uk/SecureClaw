# Claude Code Prompts — SecureClaw

Use these prompt templates when starting each Claude Code session. Always start with the **Session Opener**, then paste the relevant module prompt.

---

## Session Opener (paste this EVERY session, before any module prompt)

```
I am building SecureClaw, a secure-by-design AI agent platform. This is a real production project.

Before writing any code, read and fully internalise these governing documents in order:

1. secureclaw/CLAUDE.md — security mandates (10 hard rules, no exceptions)
2. secureclaw/docs/security/SRS.md — all security requirements (R1–R8)
3. secureclaw/docs/architecture/ARCHITECTURE.md — full system architecture and data flows
4. secureclaw/DEPENDENCIES.md — pinned dependency versions and excluded packages
5. secureclaw/PROJECT_PLAN.md — current stage and milestone we are working on

Key rules you must follow throughout this session:
- Never generate auth bypass patterns, default credentials, or "skip in dev" shortcuts
- Never use any package on the DEPENDENCIES.md excluded list
- All secrets must use SecretRef (vault:// or env://) — never hardcoded
- All inputs must be validated with Zod (TypeScript) or Pydantic v2 (Python) before use
- If any instruction conflicts with SRS requirements, flag the conflict before writing code
- Generate unit tests alongside every module — not as an afterthought

Confirm you have read all five documents before proceeding.
```

---

## STAGE 1 PROMPTS

---

### Milestone 1.2 — Vault Integration  
**(Start here — everything else depends on secrets being available)**

```
We are implementing Milestone 1.2: Vault Integration.

Read before starting:
- secureclaw/docs/architecture/decisions/ADR-002-vault.md — decision to use Vault
- secureclaw/docs/architecture/decisions/ADR-004-vault-unseal.md — Transit auto-unseal design
- secureclaw/docs/architecture/decisions/ADR-007-audit-signing.md — Vault Transit for Ed25519
- secureclaw/docs/runbooks/VAULT_SETUP.md — the init runbook this code must support

Task: Implement the Vault adapter module at `secureclaw/vault-adapter/`.

This is a TypeScript module used by both the Gateway and the Audit Logger.

--- DELIVERABLE 1: SecretRef type and resolver ---

Create `secureclaw/vault-adapter/src/secret-ref.ts`:

```typescript
// SecretRef is the ONLY way secrets are passed around in SecureClaw.
// A SecretRef is never the secret itself — it is a pointer to where the secret lives.
export type SecretRef =
  | { type: 'vault'; path: string; field: string }   // vault://secret/data/secureclaw/llm#anthropic_api_key
  | { type: 'env';   name: string }                   // env://SECURECLAW_SESSION_SECRET

// Implement:
export function parseSecretRef(uri: string): SecretRef  // parse the URI string format
export async function resolveSecretRef(ref: SecretRef, client: VaultClient): Promise<string>  // resolve to actual value
```

Rules:
- resolveSecretRef must NEVER log or expose the resolved value
- The resolved string must never appear in any TypeScript variable named in a way that reveals it (name it 'resolved' or 'value', not 'password', 'token', 'apiKey' etc.)
- Resolution must fail loudly (throw) if Vault is unreachable — no silent fallbacks

--- DELIVERABLE 2: VaultClient class ---

Create `secureclaw/vault-adapter/src/vault-client.ts`:

Implement class VaultClient with:
- Constructor takes: vaultAddr (string), vaultToken (SecretRef pointing to env://) — NOT a raw token
- `kv.get(path: string, field: string): Promise<string>` — reads KV v2
- `transit.sign(keyName: string, payloadBase64: string): Promise<string>` — calls Transit sign endpoint, returns base64 signature
- `transit.exportPublicKey(keyName: string): Promise<string>` — exports public key PEM
- Token auto-renewal: check token TTL every 5 minutes, renew if TTL < 300s
- On Vault error: exponential backoff (1s, 2s, 4s, max 30s), then throw after 3 retries
- Use node-vault package (^0.10.0 — already in DEPENDENCIES.md)

--- DELIVERABLE 3: Update docker-compose.yml ---

Update `secureclaw/deploy/docker-compose.yml` to add:
1. `vault-transit` service:
   - Image: hashicorp/vault:1.18.3
   - Backend network only (NOT exposed to host)
   - Port 8210 (internal only)
   - Config: minimal — Transit secrets engine only
   - Healthcheck: vault status
2. `qdrant` service:
   - Image: qdrant/qdrant:1.13.2
   - Backend network only
   - Port 6333 (internal only)
   - Volume: qdrant_data (persistent)
   - Environment: QDRANT__SERVICE__API_KEY via SecretRef (env://)
   - Read-only filesystem except /qdrant/storage
3. Update `vault` service to use Transit auto-unseal:
   - VAULT_SEAL_TYPE=transit
   - VAULT_TRANSIT_SEAL_KEY_NAME=autounseal
   - VAULT_TRANSIT_SEAL_ADDRESS=http://vault-transit:8210
   - VAULT_TRANSIT_SEAL_TOKEN from secrets file mount
   - Depends on: vault-transit (healthy)

All services: non-root user, read-only filesystem where possible, no privileged mode, resource limits (memory: 512m for vault-transit, 1g for vault, 512m for qdrant).

--- DELIVERABLE 4: first-run.sh ---

Create `secureclaw/deploy/first-run.sh`:

A bash script that a new operator runs ONCE to initialise the full stack from cold.

Script must:
1. Check prerequisites (docker, docker compose, openssl, jq, vault CLI)
2. Create `secureclaw/deploy/secrets/` directory (chmod 700)
3. Add `secureclaw/deploy/secrets/` to .gitignore if not already present
4. Start vault-transit only, wait for healthy
5. Init vault-transit (1-of-1 shares), save unseal key + root token to secrets/ with chmod 600
6. Unseal vault-transit
7. Configure vault-transit: enable transit, create autounseal key, create autounseal policy, generate autounseal token
8. Start vault, wait for healthy (auto-unseal should work)
9. Init primary vault (5-of-3 recovery), save recovery keys + root token to secrets/ with chmod 600
10. Configure primary vault: enable KV v2, enable Transit, create audit-signing ed25519 key
11. Seed all secrets (prompt operator for API keys, generate random values for internal secrets)
12. Create service policies and tokens, write to secrets/ files
13. Start full stack (`docker compose up -d`)
14. Run healthcheck on all services
15. Print summary: Vault status, services healthy, next steps

Script must:
- Fail immediately on any error (set -euo pipefail)
- Print clear progress messages with colour (green=ok, red=error, yellow=warning)
- Be idempotent where possible (check if already initialised before re-running steps)
- Include a --rotate-tokens flag that only re-runs steps 12–13
- Never print secret values to stdout — write directly to files

--- DELIVERABLE 5: Unit tests ---

Create `secureclaw/vault-adapter/src/__tests__/`:

1. `secret-ref.test.ts`: test parseSecretRef with valid and invalid URIs
2. `vault-client.test.ts`: mock node-vault, test KV get, Transit sign, token renewal, retry backoff, error handling

Use vitest (^3.0.9). Mock all network calls — no real Vault required for unit tests.

--- SECURITY REVIEW GATE ---

After generating all code, review your own output against these checks before presenting it:
[ ] No raw secret values in any variable, log, or error message
[ ] SecretRef used everywhere a secret is needed
[ ] VaultClient throws (not returns null/undefined) on all failure paths
[ ] docker-compose.yml has no plaintext secrets — all via secrets file mounts or env refs
[ ] first-run.sh uses set -euo pipefail
[ ] first-run.sh writes secrets to files with chmod 600, never to stdout
[ ] All test mocks verify error paths, not just happy paths

If any check fails, fix it before presenting the code.
```

---

### Milestone 1.1 — Gateway Skeleton
**(Run after Vault is operational)**

```
We are implementing Milestone 1.1: TypeScript Gateway Skeleton.

Vault is already running and the vault-adapter module is complete at `secureclaw/vault-adapter/`.

Read before starting:
- secureclaw/docs/architecture/decisions/ADR-001-gateway.md — TypeScript gateway decision
- secureclaw/docs/architecture/decisions/ADR-006-ipc-protocol.md — Unix socket IPC spec
- secureclaw/DEPENDENCIES.md — use Hono (^4.7.4), jose (^5.9.6), zod (^3.24.1), rate-limiter-flexible (^5.0.4), pino (^9.6.0)

Task: Scaffold and implement the Gateway at `secureclaw/gateway/`.

--- DELIVERABLE 1: Project scaffold ---

Create `secureclaw/gateway/` as a pnpm TypeScript project:
- package.json with all deps from DEPENDENCIES.md (TypeScript section)
- tsconfig.json: strict: true, noUncheckedIndexedAccess: true, exactOptionalPropertyTypes: true
- tsdown for build
- vitest for tests

--- DELIVERABLE 2: Session type ---

Create `secureclaw/gateway/src/auth/session.ts`:

```typescript
export type Role = 'viewer' | 'analyst' | 'operator' | 'admin'

export interface Session {
  tenant_id: string     // Always present — every request is scoped to a tenant
  user_id: string
  role: Role
  expires_at: Date      // Must be checked on every request — no stale sessions
  jti: string           // JWT ID — for replay detection
}
```

--- DELIVERABLE 3: Auth module ---

Create `secureclaw/gateway/src/auth/`:

1. `jwt-validator.ts`: Validates JWTs using jose ONLY.
   - Algorithm whitelist: ['ES256', 'RS256'] — reject all others including 'none' and 'HS256'
   - Validate: exp, nbf, iss, aud, jti claims
   - Return Session on success, throw AuthError (typed) on any failure
   - JWKS caching with 5 min TTL, force refresh on unknown kid

2. `local-token.ts`: Validates local signed tokens (Phase 1, before OIDC).
   - Token = HMAC-SHA256(session_json, signing_secret) where signing_secret comes from Vault via SecretRef
   - validate(token: string): Promise<Session> — return Session or throw AuthError
   - Issue tokens via issue(session: Omit<Session, 'jti'>): Promise<string>

3. `auth-middleware.ts`: Hono middleware.
   - Reads Authorization: Bearer <token> header
   - Tries local-token first, then jwt-validator
   - On failure: return 401 JSON { error: 'Unauthorized', code: string } — no stack traces
   - On success: attach Session to Hono context (c.set('session', session))
   - Log every auth failure as structured pino event (no secret values in logs)

--- DELIVERABLE 4: Rate limiter middleware ---

Create `secureclaw/gateway/src/middleware/rate-limiter.ts`:

- Use rate-limiter-flexible (sliding window)
- Two limiters: per-IP (100 req/min) and per-user (60 req/min)
- Per-user limiter only active after auth (uses session.user_id)
- On limit exceeded: return 429 JSON { error: 'Too Many Requests', retry_after: number }
- Emit audit event on limit trigger (use AuditLogger interface — can be a stub for now)
- Config (limits) loaded from environment via Zod-validated config schema

--- DELIVERABLE 5: Zod schema validation middleware ---

Create `secureclaw/gateway/src/middleware/validate.ts`:

- Generic middleware factory: validate(schema: ZodSchema) => HonoMiddleware
- On validation failure: return 400 JSON { error: 'Bad Request', issues: ZodIssue[] }
- Strip unknown fields from validated output (do not pass unknown fields to agent)
- Max request body size: 64KB — reject with 413 if exceeded

--- DELIVERABLE 6: Channel adapters ---

Create `secureclaw/gateway/src/channels/`:

1. `cli-adapter.ts`:
   - Reads from stdin, writes to stdout
   - Each line = one JSON message: { message: string, session_token: string }
   - Validates with Zod, authenticates, forwards to IPC client
   - Streams response back to stdout

2. `websocket-adapter.ts`:
   - Hono WebSocket endpoint at /ws
   - Auth via first message (token handshake), then session attached to connection
   - Message schema: { type: 'message', content: string } | { type: 'ping' }
   - Max message size: 64KB
   - On auth failure: close connection with code 4001

--- DELIVERABLE 7: IPC client (Unix socket, per ADR-006) ---

Create `secureclaw/gateway/src/ipc/ipc-client.ts`:

Implement the gateway side of the length-prefixed JSON Unix socket protocol (ADR-006):

- Socket path: process.env.AGENT_SOCKET_PATH || '/run/secureclaw/agent.sock'
- Frame format: 4-byte big-endian uint32 length + UTF-8 JSON payload
- Max message size: 1MB — throw if exceeded
- Ping/pong heartbeat: 30s interval, reconnect on 5s timeout
- send(message: IpcMessage): Promise<IpcResponse> — correlation via id field
- IpcMessage schema validated with Zod before sending
- Auto-reconnect with exponential backoff (1s, 2s, 4s, max 30s)

--- DELIVERABLE 8: Health endpoint ---

`GET /health` returns:
```json
{ "status": "ok", "vault": "connected", "agent": "connected", "version": "0.1.0" }
```
No auth required on /health. Checks Vault connectivity and IPC socket connectivity.
Return 503 if either is down.

--- DELIVERABLE 9: Unit tests ---

Create `secureclaw/gateway/src/__tests__/`:

- `auth.test.ts`: valid token, expired token, wrong algorithm, missing claims, replay attack (duplicate jti)
- `rate-limiter.test.ts`: under limit passes, over limit returns 429, per-user vs per-IP limits
- `validate.test.ts`: valid schema passes, invalid schema returns 400, oversized body rejected, unknown fields stripped
- `ipc-client.test.ts`: message framing correct, max size enforced, reconnect on disconnect

Mock all external dependencies. Test error paths as thoroughly as happy paths.

--- SECURITY REVIEW GATE ---

After generating all code, check:
[ ] JWT algorithm whitelist in place — 'none' and 'HS256' explicitly rejected
[ ] No stack traces or internal errors returned to clients
[ ] No secret values in any pino log output
[ ] All channel adapters enforce max message size
[ ] IPC client validates message schema before sending (not after receiving)
[ ] Rate limiter active on all message-processing endpoints
[ ] /health returns 503 (not 500) when dependencies are down
[ ] All Vault access via SecretRef, not raw tokens
```

---

### Milestone 1.3 — Agent Runtime Skeleton
**(Run after Gateway is operational)**

```
We are implementing Milestone 1.3: Python Agent Runtime Skeleton.

The Gateway (secureclaw/gateway/) and Vault adapter are complete.
The agent receives messages from the Gateway via Unix socket and returns responses.

Read before starting:
- secureclaw/docs/architecture/decisions/ADR-001-gateway.md — Python agent role
- secureclaw/docs/architecture/decisions/ADR-006-ipc-protocol.md — Unix socket IPC spec (MUST implement this exactly)
- secureclaw/DEPENDENCIES.md — use litellm>=1.82.1, pydantic>=2.11.1, hvac>=2.3.0, structlog>=25.1.0, httpx>=0.28.1

Task: Scaffold and implement the Agent Runtime at `secureclaw/agent/`.

--- DELIVERABLE 1: Project scaffold ---

Create `secureclaw/agent/` as a uv Python project:
- pyproject.toml with all deps from DEPENDENCIES.md (Python section)
- All litellm, pydantic, hvac, structlog, httpx at minimum versions in DEPENDENCIES.md
- pytest + pytest-asyncio for tests
- Dockerfile.agent (already exists at secureclaw/deploy/ — reference it, do not recreate)

--- DELIVERABLE 2: IPC server (per ADR-006 spec exactly) ---

Create `secureclaw/agent/src/ipc/server.py`:

Implement the agent side of the length-prefixed JSON Unix socket protocol:

```python
# Frame format (MUST match ADR-006 exactly):
# [4 bytes: big-endian uint32 message length][N bytes: UTF-8 JSON]

class IpcServer:
    socket_path: str  # /run/secureclaw/agent.sock
    
    async def start(self) -> None: ...
    async def stop(self) -> None: ...
    async def _handle_connection(self, reader, writer) -> None: ...
    async def _read_frame(self, reader) -> dict: ...
    async def _write_frame(self, writer, message: dict) -> None: ...
```

Rules:
- Socket file created with mode 0o600 (owner only)
- Max message size: 1MB — close connection if exceeded
- All messages validated against IpcMessage Pydantic schema before processing
- tenant_id and user_id validated on every message (not just on connect)
- Unknown message types: return error response, do not crash
- Ping: respond with pong immediately
- Structured logging for every connection open/close and every message received

--- DELIVERABLE 3: IPC message schema (Pydantic v2, must match ADR-006) ---

Create `secureclaw/agent/src/ipc/schema.py`:

```python
from pydantic import BaseModel, Field
from enum import Enum
from typing import Literal, Union

class MessageType(str, Enum):
    REQUEST = 'request'
    RESPONSE = 'response'
    ERROR = 'error'
    PING = 'ping'
    PONG = 'pong'

class IpcMessage(BaseModel):
    version: Literal['1']           # Reject any other version
    id: str = Field(pattern=r'^[0-9a-f-]{36}$')  # UUID v4
    type: MessageType
    tenant_id: str = Field(min_length=1, max_length=64, pattern=r'^[a-zA-Z0-9_-]+$')
    user_id: str = Field(min_length=1, max_length=64, pattern=r'^[a-zA-Z0-9_-]+$')
    payload: dict

    model_config = {'extra': 'forbid'}  # Reject unknown fields
```

Also define RequestPayload, ResponsePayload, ErrorPayload Pydantic models.

--- DELIVERABLE 4: LiteLLM adapter ---

Create `secureclaw/agent/src/llm/adapter.py`:

```python
class LlmAdapter:
    # Supports: anthropic, openai, ollama (local, no API key)
    # Provider and model configured via environment SecretRef, not hardcoded
    
    async def complete(
        self,
        messages: list[dict],
        tools: list[dict] | None = None,
        max_tokens: int = 4096,
    ) -> LlmResponse: ...
```

Rules:
- Use litellm.acompletion (async)
- litellm version must be >=1.82.1 (enforced in pyproject.toml)
- NEVER enable custom_code_guardrail in litellm config (see DEPENDENCIES.md CVE watch list)
- Provider API keys loaded from Vault via hvac (SecretRef pattern matching TypeScript side)
- Timeout: 30s per completion, configurable via env
- On LiteLLM error: structured log + raise LlmError (typed exception) — never swallow errors
- Max tokens capped at 8192 regardless of caller request

--- DELIVERABLE 5: Agent loop ---

Create `secureclaw/agent/src/agent/loop.py`:

```python
class AgentLoop:
    max_iterations: int = 10      # Hard cap — no infinite loops
    timeout_seconds: int = 120    # Total loop timeout
    
    async def run(
        self,
        session_message: str,
        session: dict,            # tenant_id, user_id, role from IPC message
        tools: list | None = None # Empty for Milestone 1.3 — tools added in Stage 3
    ) -> str: ...
```

Rules:
- Iteration counter incremented on every LLM call — hard stop at max_iterations
- asyncio.timeout(self.timeout_seconds) wraps entire loop — hard stop on timeout
- On max iterations or timeout: return a graceful error response string, never raise to caller
- Log every iteration: iteration number, token count, structured event
- For Milestone 1.3: single-turn only (no tool calls yet) — but design the loop to support tools in Stage 3

--- DELIVERABLE 6: Vault client (Python, hvac) ---

Create `secureclaw/agent/src/vault/client.py`:

Python equivalent of the TypeScript VaultClient:
- Uses hvac (>=2.3.0)
- Reads VAULT_ADDR and VAULT_TOKEN from environment (token itself from env://) 
- kv_get(path: str, field: str) -> str
- Token renewal: check every 5 min, renew if TTL < 300s
- Retry with exponential backoff: 1s, 2s, 4s — raise after 3 failures

--- DELIVERABLE 7: Main entrypoint ---

Create `secureclaw/agent/src/main.py`:

- Parse config from environment (Pydantic Settings model, all validated)
- Initialise Vault client, resolve all SecretRefs at startup
- Start IPC server
- Handle SIGTERM/SIGINT gracefully: drain in-flight requests, close socket cleanly
- Structured startup log: all config values (no secret values), service versions

--- DELIVERABLE 8: Unit tests ---

Create `secureclaw/agent/tests/`:

- `test_ipc_framing.py`: frame encode/decode, max size enforcement, schema validation, unknown type rejection
- `test_llm_adapter.py`: mock litellm, test timeout, max token cap, error handling
- `test_agent_loop.py`: mock LlmAdapter, test max iterations hit, timeout hit, single-turn happy path
- `test_vault_client.py`: mock hvac, test kv_get, retry backoff, token renewal

All tests async (pytest-asyncio). Mock all external I/O. 100% of error paths tested.

--- INTEGRATION TEST (run after all three milestones complete) ---

Create `secureclaw/tests/integration/test_e2e_stage1.py`:

With Docker Compose stack running:
1. Start CLI adapter with a test session token
2. Send: `{ "message": "Hello, what is 2+2?", "session_token": "<valid_test_token>" }`
3. Assert: response received within 10 seconds
4. Assert: response is a non-empty string
5. Assert: audit log entry exists in Loki for this request
6. Assert: Prometheus counter `secureclaw_requests_total` incremented

This is the Stage 1 exit criteria test.

--- SECURITY REVIEW GATE ---

After generating all code, check:
[ ] IPC socket file created with mode 0o600
[ ] IpcMessage schema uses extra='forbid' (no unknown fields)
[ ] tenant_id and user_id validated on every message (pattern regex enforced)
[ ] LiteLLM version pinned to >=1.82.1 in pyproject.toml
[ ] custom_code_guardrail NOT present anywhere in litellm config
[ ] Max iterations and timeout are hard stops (cannot be overridden at runtime)
[ ] No API keys or secrets in any log output
[ ] All Vault access uses hvac with token from environment — no hardcoded values
[ ] Graceful shutdown drains in-flight requests (no dropped responses on SIGTERM)
```

---

## STAGE 2 PROMPTS

---

### Milestone 2.1 — Policy Engine

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

### Milestone 2.2 — Prompt Injection Detection

```
Implement the SecureClaw prompt injection detection system.

Read before starting:
- secureclaw/docs/architecture/decisions/ADR-008-canary-tokens.md — HMAC-SHA256 canary design (implement this exactly)
- secureclaw/docs/security/SRS.md R6.2 — 4-layer detection requirement

Generate:
1. InjectionDetector class with detect(input: str) -> DetectionResult
2. Pattern library (20+ patterns): role overrides, ignore instructions, system prompt exfil, jailbreaks
3. CanaryManager using HMAC-SHA256 design from ADR-008: inject_canary(session_id) -> str, check_output(output, session_id) -> bool
4. OutputGuard: validate_tool_call(llm_output: dict) -> ValidatedToolCall | GuardViolation
5. Session quarantine on canary trigger (audit event + alert webhook)
6. Adversarial test suite: 100+ prompt injection cases — all must be detected
```

---

### Milestone 2.3 — Audit Logger

```
Implement the SecureClaw Audit Logger (secureclaw/audit/src/).

Read before starting:
- secureclaw/docs/architecture/decisions/ADR-007-audit-signing.md — Vault Transit Ed25519 signing (implement this exactly)

Requirements from SRS:
- R7.1: Structured events for all audit points
- R7.2: Ed25519 signatures via Vault Transit (private key never leaves Vault)
- R7.3: Loki sink (hot), local file (warm)
- R4.5: Secret redaction before signing

Generate:
1. AuditEvent Zod schema
2. SecretRedactor: redact(payload: object) -> object
3. AuditLogger: log(event) — redact, canonicalise JSON, hash (SHA-256), sign via Vault Transit, emit to Loki + file
4. Verification function: verify(entry, publicKeyPem) -> boolean
5. Unit tests: redaction, signature verification, tamper detection

Note: The circular dependency (Vault access generates audit events that need Vault signing) is resolved in ADR-007. Read it before implementing.
```

---

## Codex Review — Paste With Any Module

See `secureclaw/docs/development/CODEX_REVIEW_CHECKLIST.md` — paste alongside any completed module for independent security review.
