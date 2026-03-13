# Security Policy — SecureClaw

## Supported Versions

Only the latest release receives security patches.

## Reporting a Vulnerability

Please **do not** open a public GitHub issue for security vulnerabilities.

Report privately via GitHub Security Advisories:
👉 https://github.com/zebadee2kk/openclaw/security/advisories/new

Include:
- Description of the vulnerability
- Steps to reproduce
- Affected versions
- Potential impact
- Any suggested mitigations

We aim to acknowledge reports within **48 hours** and provide an initial assessment within **7 days**.

## Security Design Principles

SecureClaw is built secure-by-design. Core principles:

1. **Zero trust** — every request authenticated and authorised at every layer
2. **Least privilege** — components and tools have minimum required permissions only
3. **Secure defaults** — dangerous capabilities are opt-in, never opt-out
4. **Defence in depth** — multiple independent layers for every threat scenario
5. **Immutable audit trail** — all actions signed and tamper-evident
6. **Sandbox everything** — tool execution isolated in containers per invocation

See `docs/security/SRS.md` for the full Security Requirements Specification.

## Disclosure Policy

We follow coordinated disclosure. Security fixes are released before full public disclosure.
