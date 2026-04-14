# OMA LwM2M Expert Skill for Claude, OpenClaw, OpenAI, ChatGPT, xAI, Gemini and more...

A comprehensive OMA LwM2M (Lightweight Machine-to-Machine) skill that turns Claude into a senior IoT protocol engineer — covering all specification versions from v1.0 (2017) through v2.0 (anticipated 2026), the complete object/resource data model, all transport bindings, security architecture, and the surrounding implementation ecosystem.

<p align="center">
  <img src="https://lwm2m.openmobilealliance.org/wp-content/uploads/2024/05/LwM2M-logo.png" alt="LwM2M Logo" width="300">
</p>

## What It Does

This skill gives Claude deep, standards-grounded expertise across the full OMA LwM2M ecosystem:

* **All versions**: v1.0, v1.0.2, v1.1, v1.1.1, v1.2, v1.2.1, v1.2.2, v2.0 (anticipated)
* **Protocol stack**: CoAP (RFC 7252), DTLS 1.2/1.3, TLS 1.2/1.3, OSCORE, SenML, CBOR
* **All interfaces**: Bootstrap, Registration, DM&SE, Information Reporting
* **All transports**: UDP, TCP, SMS, Non-IP (3GPP CIoT, LoRaWAN), MQTT, HTTP
* **Security**: PSK, RPK, x509, EST, OSCORE, DTLS CID (RFC 9146), session persistence
* **Data model**: Core objects (0-25), IPSO Smart Objects (3300+), OMNA registry, object versioning
* **Implementation**: Wakaama, Leshan, Anjay, Californium, Zephyr LwM2M, DTLS library comparison
* **Deployment**: Queue Mode, NB-IoT/LTE-M PSM, firmware update (FOTA), gateway architecture
* **Ecosystem**: 3GPP CIoT integration, LoRaWAN, cloud platforms, industry verticals

## Example Questions It Handles

**Specification deep-dives:**

> "Walk me through how the Bootstrap-Pack-Request works in v1.2.2 and how it differs from client-initiated bootstrap."

**Security architecture:**

> "Explain how DTLS CID (RFC 9146) solves the NAT rebinding problem for NB-IoT devices in PSM. Show the message flow."

**Implementation guidance:**

> "What are the trade-offs between using mbedTLS vs TinyDTLS for a Wakaama-based client on nRF9161?"

**Cross-version comparison:**

> "Compare the data encoding options across LwM2M versions — TLV, JSON, SenML-JSON, SenML-CBOR, LwM2M CBOR."

**Deployment planning:**

> "We're deploying 50,000 NB-IoT water meters. Walk me through the LwM2M architecture — Queue Mode, DTLS CID, firmware update strategy, and lifetime/pmax tuning."

**Object model design:**

> "How do I register a new LwM2M Object with the OMNA registry? What are the rules for reusable resources?"

## Installation

### Claude Desktop (Cowork)

1. Download `oma-lwm2m-expert.skill` from [Releases](https://github.com/YOUR-ORG/oma-lwm2m-expert/releases)
2. Open it — Claude will prompt you to install

### Manual Install

Copy the `oma-lwm2m-expert/` folder into your Claude skills directory:

```
~/.claude/skills/oma-lwm2m-expert/
├── SKILL.md
└── references/
    ├── versions.md
    ├── objects.md
    ├── protocol-details.md
    ├── security.md
    ├── implementations.md
    └── ecosystem.md
```

### Claude Project

Upload the skill files to a Claude Project's knowledge base, or add the SKILL.md as a project instruction file with the references/ folder as supplementary files.

## What's Inside

| File | Purpose |
|------|---------|
| `SKILL.md` | Main skill instructions — response patterns, 6 knowledge domains, when to web search |
| `references/versions.md` | Detailed version-by-version breakdown (v1.0 → v2.0) with feature deltas |
| `references/objects.md` | Complete core objects reference (0-25), IPSO Smart Objects, OMNA registry |
| `references/protocol-details.md` | CoAP mapping, message flows, transport bindings, Wireshark patterns |
| `references/architecture.md` | Complete protocol flows (Bootstrap/Registration/Observe/FOTA/SOTA), client HAL/PAL architecture, server architecture, NBI/hyperscaler integration, massive-scale IoT patterns, production checklist |
| `references/security.md` | DTLS/TLS/OSCORE deep-dive, CID flow, credential provisioning, library comparison |
| `references/implementations.md` | Open-source stacks (Wakaama, Leshan, Anjay), testing, interop notes |
| `references/ecosystem.md` | Gateway, 3GPP CIoT, LoRaWAN, cloud platforms, industry verticals, v2.0 roadmap |

## Coverage

### Specification Versions

| Version | Published | Key Additions |
|---------|-----------|---------------|
| v1.0 | Feb 2017 | Core protocol: Bootstrap, Registration, DM&SE, Observe, Queue Mode |
| v1.0.2 | Feb 2018 | Bug fixes and errata |
| v1.1 | Jun 2018 | TCP/TLS, Non-IP (3GPP CIoT, LoRaWAN), OSCORE, SenML |
| v1.1.1 | Jun 2019 | CBOR for single resources |
| v1.2 | Nov 2020 | MQTT, HTTP, Gateway, Composite ops, Send, LwM2M CBOR, DTLS CID |
| v1.2.1 | Dec 2022 | Interop clarifications |
| v1.2.2 | Jun 2024 | Bug fixes, editorial improvements |
| v2.0 | 2026 (est.) | Profile IDs, delta FOTA, eSIM, edge computing proxy |

### Underlying RFCs

| RFC | Title | LwM2M Role |
|-----|-------|------------|
| RFC 7252 | CoAP | Application protocol |
| RFC 7641 | CoAP Observe | Notification mechanism |
| RFC 7959 | CoAP Blockwise | Large payload transfer |
| RFC 8323 | CoAP over TCP/TLS/WS | TCP transport binding |
| RFC 6347 | DTLS 1.2 | UDP transport security |
| RFC 9146 | DTLS Connection ID | NAT traversal for sleepy devices |
| RFC 9147 | DTLS 1.3 | Modern transport security |
| RFC 8613 | OSCORE | Application-layer security |
| RFC 8428 | SenML | Data encoding |
| RFC 8949 | CBOR | Binary encoding |

## Complementary Skills

This skill pairs well with:
- **[3GPP Expert Skill](https://github.com/lugasia/3gpp-skill)** — For 3GPP radio access and core network context (NB-IoT, LTE-M, 5G CIoT)

## Contributing

Contributions welcome! If you spot an inaccuracy, want to add coverage for a specific topic, or have suggestions:

1. Open an issue describing the improvement
2. Fork the repo and submit a PR
3. Ensure all specification references are accurate (cite OMA-TS document and section)
4. Use correct OMA/IETF terminology

## License

MIT License — see [LICENSE](LICENSE) for details.

---

Built by Sean van der Walt — [Walt Technologies](https://github.com/svdwalt007)
