# Security Policy

## Scope

This repository contains documentation and skill definitions for the OMA LwM2M Expert Skill. It does not contain executable code, but we take the integrity of specification content seriously — inaccurate security guidance could lead to insecure IoT deployments.

## Reporting a Vulnerability

If you discover a security-relevant inaccuracy in our content (e.g., incorrect DTLS configuration guidance, outdated security mode recommendations, or misleading credential provisioning instructions), please report it responsibly:

1. **Email**: Open a private security advisory via [GitHub Security Advisories](https://github.com/svdwalt007/Best-LwM2M-Agentic-Skills/security/advisories/new)
2. **Alternative**: Open a [Correction issue](https://github.com/svdwalt007/Best-LwM2M-Agentic-Skills/issues/new?template=correction.md) with the `[SECURITY]` prefix

## Response Timeline

- **Acknowledgement**: Within 48 hours
- **Assessment**: Within 7 days
- **Correction**: Critical security inaccuracies will be prioritised and corrected as soon as possible

## Supported Versions

| Version | Supported |
|---------|-----------|
| Latest (main branch) | Yes |
| Older releases | Best effort |

## Security Content Areas

The following reference files contain security-critical content that receives priority review:

- `references/security.md` — DTLS/TLS/OSCORE configurations, credential provisioning, security modes
- `references/architecture.md` — Bootstrap flows, credential management architecture
- `references/protocol-details.md` — Transport security bindings
- `SKILL.md` — Security-related response patterns
