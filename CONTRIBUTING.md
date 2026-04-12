# Contributing to OMA LwM2M Expert Skill

Thank you for your interest in improving the world's most comprehensive LwM2M skill for Claude! This guide will help you contribute effectively.

## How to Contribute

### Reporting Issues

- **Specification inaccuracy?** Use the [Correction template](https://github.com/svdwalt007/Best-LwM2M-Agentic-Skills/issues/new?template=correction.md)
- **Missing coverage?** Use the [Coverage Request template](https://github.com/svdwalt007/Best-LwM2M-Agentic-Skills/issues/new?template=coverage-request.md)
- **General issue?** [Open a blank issue](https://github.com/svdwalt007/Best-LwM2M-Agentic-Skills/issues/new)

### Submitting Changes

1. **Fork** the repository
2. **Create a branch** from `main` with a descriptive name (e.g., `fix/dtls-cid-flow`, `add/mqtt-transport-details`)
3. **Make your changes** following the guidelines below
4. **Test your changes** by loading the skill into Claude and verifying responses
5. **Submit a Pull Request** with a clear description of what changed and why

### Content Guidelines

#### Accuracy

- All specification references must cite the exact **OMA-TS document**, **section number**, and **version**
- RFC references should include the RFC number and relevant section
- When in doubt, link to the [OMA LwM2M Registry](https://technical.openmobilealliance.org/OMNA/LwM2M/LwM2MRegistry.html)

#### Terminology

- Use official **OMA/IETF terminology** consistently
- "LwM2M" (not "LWM2M" or "Lwm2m")
- "Bootstrap-Server" (hyphenated, capitalised)
- "DM&SE" for Device Management and Service Enablement
- "DTLS CID" (not "DTLS connection ID" in headings)

#### Style

- Use Markdown tables for structured comparisons
- Use code blocks for message flows and CoAP examples
- Keep explanations grounded in specifications, not opinions
- Prefer precision over brevity — engineers need exact details

### File Structure

| File | What goes here |
|------|---------------|
| `SKILL.md` | Skill definition, response patterns, knowledge domain summaries |
| `references/versions.md` | Version-by-version feature deltas |
| `references/objects.md` | Object/resource data model details |
| `references/protocol-details.md` | CoAP mapping, message flows, transport bindings |
| `references/security.md` | DTLS/TLS/OSCORE, credential provisioning |
| `references/implementations.md` | Open-source stacks, tools, interop |
| `references/architecture.md` | Protocol flows, client/server architecture, deployment |
| `references/ecosystem.md` | Gateway, 3GPP, LoRaWAN, cloud, verticals |

### What Makes a Great Contribution

- Fills a gap identified by real-world usage (e.g., "Claude couldn't answer X")
- Includes authoritative citations
- Follows the existing file structure and formatting conventions
- Improves Claude's ability to reason about LwM2M, not just recite facts

## Code of Conduct

Please review our [Code of Conduct](CODE_OF_CONDUCT.md) before contributing.

## Questions?

Open a [discussion](https://github.com/svdwalt007/Best-LwM2M-Agentic-Skills/issues) or reach out via the issue tracker.

---

Thank you for helping make LwM2M expertise more accessible to engineers worldwide!
