# Fork Management Notes

This document records how `zebadee2kk/openclaw` (SecureClaw) is managed relative to the upstream `openclaw/openclaw` project.

---

## Branch Model

| Branch | Purpose | Synced from upstream? |
|---|---|---|
| `main` | Fork baseline, clean sync point | ✅ Yes — periodically |
| `secureclaw` | Active SecureClaw development | ❌ No — cherry-pick only |
| `release/v*` | Tagged stable SecureClaw releases | ❌ No |

**Rule:** Never merge upstream directly into `secureclaw`. The SecureClaw architecture diverges sufficiently that blind merges will cause conflicts and may introduce regressions in security controls.

---

## Syncing `main` from Upstream

Run periodically (monthly, or when a significant upstream release lands):

```bash
# One-time setup (only needed once)
git remote add upstream https://github.com/openclaw/openclaw.git

# Sync main
git checkout main
git fetch upstream
git merge upstream/main
git push origin main
```

Review the upstream changelog before merging — check for:
- Breaking changes to the Gateway protocol
- Security-relevant changes (DM policy, sandbox model, auth)
- New channel integrations you may want to cherry-pick into `secureclaw`

---

## Cherry-Picking Upstream Changes into `secureclaw`

When a specific upstream commit is relevant to SecureClaw (bug fix, new feature, security patch):

```bash
git checkout secureclaw
git cherry-pick <commit-sha>
```

Always test after cherry-picking. Vault integration and SecretRef resolution are the most likely conflict points.

---

## What to Pull vs. What to Freeze

### ✅ Generally safe to pull
- New channel integrations (WhatsApp, Telegram, etc.)
- CLI improvements and bug fixes
- Model failover and session improvements
- Documentation updates

### ⚠️ Review carefully before pulling
- Docker / compose changes (may conflict with SecureClaw hardening)
- Auth model changes (`dmPolicy`, `allowFrom`, sandbox config)
- Dependency upgrades (check for CVEs first)

### ❌ Do not pull without deliberate SecureClaw review
- Changes to secret/credential handling
- Gateway WebSocket protocol breaking changes
- Any change touching `credentials/` or token storage

---

## Upstream Project Links

- Repo: https://github.com/openclaw/openclaw
- Docs: https://docs.openclaw.ai
- Releases: https://github.com/openclaw/openclaw/releases
- Changelog: https://github.com/openclaw/openclaw/blob/main/CHANGELOG.md

---

## SecureClaw Development Stages

| Stage | Focus | Status |
|---|---|---|
| Stage 1 | Vault integration, SecretRef, hardened compose, first-run script | 🔄 In Progress |
| Stage 2 | Prompt injection hardening, agent sandboxing, audit logging | ⏳ Planned |
| Stage 3 | mTLS, secrets rotation automation, compliance controls | ⏳ Planned |

For Claude Code session prompts covering each milestone, see [`secureclaw/docs/development/CLAUDE_CODE_PROMPTS.md`](secureclaw/docs/development/CLAUDE_CODE_PROMPTS.md).
