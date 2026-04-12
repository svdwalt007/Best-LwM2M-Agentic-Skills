# OMA LwM2M Specification Versions — Detailed Reference

## Table of Contents
1. [Version Timeline](#version-timeline)
2. [v1.0 — Initial Release](#v10--initial-release-february-2017)
3. [v1.0.2 — Errata](#v102--errata-february-2018)
4. [v1.1 — Non-IP & TCP](#v11--non-ip--tcp-june-2018)
5. [v1.1.1 — CBOR Enhancement](#v111--cbor-enhancement-june-2019)
6. [v1.2 — Gateway & Composite Ops](#v12--gateway--composite-operations-november-2020)
7. [v1.2.1 — Clarifications](#v121--clarifications-december-2022)
8. [v1.2.2 — Bug Fixes](#v122--bug-fixes-june-2024)
9. [v2.0 — Anticipated](#v20--anticipated-2026)
10. [Specification Documents per Version](#specification-documents-per-version)
11. [Backward Compatibility Rules](#backward-compatibility-rules)
12. [3GPP Cross-References](#3gpp-cross-references)

---

## Version Timeline

| Version | Published | Status | Key Theme |
|---------|-----------|--------|-----------|
| v1.0 | Feb 2017 | Approved | Foundational protocol |
| v1.0.2 | Feb 2018 | Approved | Bug fixes and clarifications |
| v1.1 | Jun 2018 | Approved | TCP/TLS, Non-IP, OSCORE |
| v1.1.1 | Jun 2019 | Approved | CBOR single-resource support |
| v1.2 | Nov 2020 | Approved | MQTT, HTTP, Gateway, Composite ops |
| v1.2.1 | Dec 2022 | Approved | Errata, interop fixes |
| v1.2.2 | Jun 2024 | Approved | Bug fixes, editorial |
| v2.0 | Q1 2026 (est.) | In development | Profile IDs, delta FOTA, eSIM |

---

## v1.0 — Initial Release (February 2017)

The foundational specification establishing the LwM2M protocol architecture.

### Features
- **Four interfaces:** Bootstrap, Registration, Device Management & Service Enablement, Information Reporting
- **Transport bindings:** CoAP over UDP (with/without DTLS), CoAP over SMS (with/without DTLS)
- **Queue Mode:** Server queues operations while client is sleeping; client polls on registration update
- **Security modes:** PSK (0), RPK (1), x509 (2), NoSec (3)
- **Data formats:** TLV (11542), JSON (11543), Plain Text (0), Opaque (42), Core Link Format (40)
- **Core Objects (0–7):**
  - 0: LwM2M Security — server URIs, credentials, security mode
  - 1: LwM2M Server — lifetime, binding, notification storage
  - 2: Access Control — per-object ACLs for multi-server
  - 3: Device — manufacturer, model, serial, battery, memory, error codes
  - 4: Connectivity Monitoring — bearer type, signal strength, IP addresses, cell ID
  - 5: Firmware Update — package URI, state machine, update result
  - 6: Location — latitude, longitude, altitude, velocity, timestamp
  - 7: Connectivity Statistics — TX/RX data, max message size, collection period
- **Bootstrap methods:** Factory Bootstrap, Smartcard Bootstrap, Client-Initiated Bootstrap, Server-Initiated Bootstrap
- **Observation:** CoAP Observe (RFC 7641) with pmin/pmax/gt/lt/st attributes
- **Blockwise transfer:** RFC 7959 for large payloads (firmware images)

### Specification Documents
- OMA-TS-LightweightM2M-V1_0 (single document, not yet split into Core/Transport)
- OMA-ERELD-LightweightM2M-V1_0

---

## v1.0.2 — Errata (February 2018)

Bug fixes and editorial corrections to v1.0. No new features.

---

## v1.1 — Non-IP & TCP (June 2018)

Major expansion of transport bindings and security options.

### New Features
- **CoAP over TCP/TLS (RFC 8323):** NAT-friendly persistent connections; eliminates DTLS handshake overhead for always-connected devices
- **Non-IP transports:**
  - 3GPP CIoT: NB-IoT Data-over-NAS (DoNAS) and User Plane CIoT optimisation
  - LoRaWAN: CoAP mapping over LoRa Class A/C
- **OSCORE (RFC 8613):** Application-layer CoAP security independent of transport. Survives proxies, supports multicast. Uses COSE for encryption.
- **SenML data formats:**
  - SenML-JSON (Content-Format 110): JSON-based, human-readable
  - SenML-CBOR (Content-Format 112): Binary, highly efficient for constrained devices
- **Resource Instance level access:** Enables reading/writing individual resource instances within multi-instance resources
- **Enhanced bootstrapping:** Incremental bootstrap updates, improved PKI support
- **Extended registration:** Enhanced registration sequence mechanisms
- **New data types:** Unsigned Integer, Corelnk, Objlnk improvements
- **Performance:** Multi-object read/write optimizations via SenML

### Specification Structure Change
Starting with v1.1, the specification is split into two documents:
- **Core TS** — messaging layer, data model, interfaces, operations
- **Transport TS** — transport-specific mappings, security configurations

### New Objects
- Object 10: Cellular Connectivity
- Object 11: APN Connection Profile
- Object 12: WLAN Connectivity
- Object 13: Bearer Selection
- Object 14: Software Management
- Object 15: DevCapMgmt (Device Capability Management)
- Object 16: Portfolio
- Object 18: Non-IP Data Delivery (NIDD)
- Object 19: BinaryAppDataContainer
- Object 20: Event Log

---

## v1.1.1 — CBOR Enhancement (June 2019)

### New Features
- **CBOR for single resources (Content-Format 60):** Allows CBOR encoding for Read and Write operations on individual resources. Previously CBOR was only available via SenML-CBOR for composite operations.

---

## v1.2 — Gateway & Composite Operations (November 2020)

The largest feature release since v1.0. Significantly extends LwM2M's reach and efficiency.

### New Features

**Transports:**
- **MQTT transport binding:** LwM2M messaging over MQTT pub/sub. Defined in Transport TS §6.7. New MQTT Server Object (22).
- **HTTP transport binding:** LwM2M over HTTP/HTTPS. Defined in Transport TS §6.8.

**Operations:**
- **Composite Read (Read-Composite):** Read multiple resources across different objects in a single request. Uses SenML or LwM2M CBOR.
- **Composite Write (Write-Composite):** Write multiple resources across different objects in a single request.
- **Composite Observe (Observe-Composite):** Observe multiple resource paths in a single observation.
- **Send operation:** Device-initiated data push — the client sends data to the server without a prior Read request. Useful for event-driven telemetry and "fire and forget" reporting.

**Data Formats:**
- **LwM2M CBOR:** A new CBOR-based encoding optimised for the LwM2M data model. More compact than SenML-CBOR for LwM2M-specific payloads.

**Gateway:**
- **LwM2M Gateway support:** Formalised gateway architecture allowing:
  - Non-LwM2M devices to be managed through a LwM2M gateway
  - LwM2M devices behind a gateway to be proxied to the server
  - New Objects 23 (Gateway) and 24 (Gateway Routing)

**Notification Attributes (new):**
- `edge` — Trigger notification on rising/falling edge (boolean threshold crossing)
- `con` — Confirmable notifications (CON instead of NON CoAP messages)
- `hqmax` — Maximum Historical Queue depth for time-series buffering

**Security:**
- **DTLS CID (RFC 9146):** Connection ID support for DTLS 1.2. Critical for sleepy device NAT traversal.
- **DTLS 1.3 (RFC 9147):** Optional support for the latest DTLS version.
- **TLS 1.3 (RFC 8446):** For TCP transport bindings.
- **EST over CoAP:** Security Mode 4. Enrollment over Secure Transport for dynamic certificate provisioning.
- **SNI (RFC 6066):** Required for certificate-based (x509) security mode.
- **Extended Master Secret (RFC 7627):** Recommended for DTLS 1.2 deployments.

**Firmware Update:**
- Enhanced firmware update with linked instances for multi-component updates
- Pull and push delivery methods

**Bootstrap:**
- **Bootstrap-Pack-Request:** Client requests a complete bootstrap configuration package in a single exchange.

**Other:**
- Clarified object versioning rules
- Updated Endpoint Client Name URN format recommendations
- New Object 21 (OSCORE), Object 25 (LwM2M COSE)

### New Objects
- Object 21: OSCORE Security Context
- Object 22: MQTT Server
- Object 23: LwM2M Gateway
- Object 24: LwM2M Gateway Routing
- Object 25: LwM2M COSE

---

## v1.2.1 — Clarifications (December 2022)

### Changes
- Editorial clarifications across Core and Transport specifications
- Interoperability improvements identified through TestFest events
- Minor normative fixes for edge cases in composite operations
- Improved examples for SenML and LwM2M CBOR encoding
- Backward compatible with v1.2

---

## v1.2.2 — Bug Fixes (June 2024)

### Changes
- Added optional parameter to Bootstrap-Pack-Request operation
- Fixed potential ambiguity in LwM2M CBOR Create operation
- Rewrote notification attribute text for clarity and simplification
- Clarified that Execute operation arguments must be unique
- Clarified LwM2M COSE Object exclusion from Register operation parameters
- Clarified COSE and MQTT Server objects in Bootstrap-Discover responses
- Registered LwM2M CBOR content format number with IANA
- Consolidated Endpoint Client Name URN format recommendations
- Moved Attributes section to Identifiers and Resources section
- Backward compatible with v1.2 and v1.2.1

---

## v2.0 — In Development (Target: v0.9 Candidate December 2026)

Under active development by the DMSO IoT Working Group at OMA SpecWorks. Per the OMA Plenary (Kraków, April 2026), full triage of ~70 issues is complete and work is organised on GitHub with team leads per major technical area.

### Confirmed Features (per OMA Plenary April 2026)

- **Profile IDs:** Standardised device-class templates defining mandatory/optional objects for specific verticals (smart meters, trackers, sensors, gateways). Reduces device registration to a **4-byte integer** — the Profile ID instantly informs the server of the device's capabilities, expected resources, and required configuration. Critical for mass deployments (onboarding tens of thousands of meters/sensors). Eliminates guesswork and accelerates provisioning.
- **Simplified Bootstrapping:** ~25% reduction in bootstrap traffic for caDM (constrained Application DM), B-IoT (Broadband IoT), and REDCap devices. Air-time savings at scale for LPWAN deployments.
- **Advanced Firmware Update Enabler:** Push/pull delivery methods AND application-layer updates with independently managed app layers. Devices can update specific software components without full firmware reflash. Delta updates (differential patching) reduce download size by 60-90%.
- **Enhanced eSIM Remote Provisioning:**
  - Object 504: RSP (Remote SIM Provisioning) management
  - Object 3443: Enhanced eSIM/eUICC lifecycle management
  - Integration with GSMA SGP.32 bootstrap flow
- **Edge Computing Proxy Support:** Distributed architectures for low-latency local processing. Edge proxies cache, aggregate, and filter LwM2M messages. QoS-aware scheduling.
- **DTLS 1.3 alongside DTLS 1.2:** Full support for RFC 9147 while maintaining backward compatibility with DTLS 1.2 deployments.
- **QoS-aware Services:** Differentiated service levels for different object/resource observations.
- **Digital Twinning:** Smart Cities WG active on digital twinning in parallel with DMSO v2.0 work.

### Northbound API (Companion Specification)
A new companion specification being developed in a separate WG (reconvening with dedicated call cadence in 2026):
- **Purpose:** Standardises the API layer **above** any compliant LwM2M server — a common conversation layer between LwM2M servers and back-end systems (OSS/BSS, cloud platforms, enterprise IT).
- **Problem solved:** Today, back-end systems must integrate uniquely with each LwM2M server vendor. NB API eliminates this fragmentation.
- **Status:** v1.0 nearing feature-complete (April 2026). Strong pull from smart city platforms, utilities modernising AMI, and grid-edge deployments.
- **Value:** Implementers write back-end integration once — Itron, AVSystem, Paradox, Friendly, or any compliant server just works.
- **Collaboration links:** GSMA re-engagement, Camara API alignment.

### Roadmap
- **2026 H1:** ~70 issues triaged, GitHub work organised with team leads
- **2026 H2:** Target LwM2M 2.0 candidate (v0.9) release by December 2026
- **2026–2027:** ETS updates (AVSystem-driven), conformance and validation roadmap
- **2027+:** Full v2.0 approval, vertical certification programmes

### Status
Work is organised on GitHub with open contributions. Engage now — scope is still open; early contributors shape the outcome. Contact: OMA portal or DMSO WG chair.

---

## Feature Matrix by Version

| Feature | v1.0 | v1.0.2 | v1.1 | v1.1.1 | v1.2 | v1.2.1 | v1.2.2 | v2.0 |
|---------|:----:|:------:|:----:|:------:|:----:|:------:|:------:|:----:|
| **Transports** | | | | | | | | |
| CoAP/UDP (DTLS) | Y | Y | Y | Y | Y | Y | Y | Y |
| CoAP/SMS | Y | Y | Y | Y | Y | Y | Y | Y |
| CoAP/TCP (TLS) | - | - | Y | Y | Y | Y | Y | Y |
| Non-IP (3GPP CIoT) | - | - | Y | Y | Y | Y | Y | Y |
| Non-IP (LoRaWAN) | - | - | Y | Y | Y | Y | Y | Y |
| MQTT | - | - | - | - | Y | Y | Y | Y |
| HTTP | - | - | - | - | Y | Y | Y | Y |
| **Security** | | | | | | | | |
| PSK / RPK / x509 | Y | Y | Y | Y | Y | Y | Y | Y |
| OSCORE (RFC 8613) | - | - | Y | Y | Y | Y | Y | Y |
| DTLS CID (RFC 9146) | - | - | - | - | Y | Y | Y | Y |
| DTLS 1.3 (RFC 9147) | - | - | - | - | Y | Y | Y | Y |
| EST over CoAP | - | - | - | - | Y | Y | Y | Y |
| SNI required (x509) | - | - | - | - | Y | Y | Y | Y |
| **Data Formats** | | | | | | | | |
| TLV | Y | Y | Y | Y | Y | Y | Y | Y |
| LwM2M JSON | Y | Y | Y | Y | Y | Y | Y | Y |
| SenML-JSON | - | - | Y | Y | Y | Y | Y | Y |
| SenML-CBOR | - | - | Y | Y | Y | Y | Y | Y |
| CBOR (single resource) | - | - | - | Y | Y | Y | Y | Y |
| LwM2M CBOR | - | - | - | - | Y | Y | Y | Y |
| **Operations** | | | | | | | | |
| Read / Write / Execute | Y | Y | Y | Y | Y | Y | Y | Y |
| Observe / Notify | Y | Y | Y | Y | Y | Y | Y | Y |
| Queue Mode | Y | Y | Y | Y | Y | Y | Y | Y |
| Read-Composite | - | - | - | - | Y | Y | Y | Y |
| Write-Composite | - | - | - | - | Y | Y | Y | Y |
| Observe-Composite | - | - | - | - | Y | Y | Y | Y |
| Send operation | - | - | - | - | Y | Y | Y | Y |
| Bootstrap-Pack-Request | - | - | - | - | Y | Y | Y | Y |
| **Architecture** | | | | | | | | |
| LwM2M Gateway | - | - | - | - | Y | Y | Y | Y |
| Edge Computing Proxy | - | - | - | - | - | - | - | Y |
| Profile IDs | - | - | - | - | - | - | - | Y |
| Delta Firmware Updates | - | - | - | - | - | - | - | Y |
| Multi-Component FOTA | - | - | - | - | - | - | - | Y |
| eSIM Provisioning Obj | - | - | - | - | - | - | - | Y |
| **Notification Attributes** | | | | | | | | |
| pmin / pmax / gt / lt / st | Y | Y | Y | Y | Y | Y | Y | Y |
| epmin / epmax | - | - | - | - | Y | Y | Y | Y |
| edge | - | - | - | - | Y | Y | Y | Y |
| con (confirmable) | - | - | - | - | Y | Y | Y | Y |
| hqmax (historical queue) | - | - | - | - | Y | Y | Y | Y |

`Y` = Supported, `-` = Not available

---

## Migration Guide: v1.0 to v1.2

### Planning Checklist

When upgrading a v1.0 deployment to v1.2, consider these areas:

#### 1. Transport & Security
- [ ] **DTLS CID:** Enable RFC 9146 CID for all sleepy devices (Queue Mode). This is the single highest-value upgrade — eliminates re-handshake after NAT rebinding.
- [ ] **CID extension type:** Ensure both client and server use extension type **54** (RFC 9146 final), not 53 (obsolete draft).
- [ ] **SNI:** If using x509 certificates, SNI is now **required** in v1.2. Update client TLS configuration.
- [ ] **Extended Master Secret:** Recommended for DTLS 1.2 (RFC 7627). Enable in DTLS library config.
- [ ] **DTLS 1.3:** Optional in v1.2. Only enable if both sides support it; maintain DTLS 1.2 fallback.
- [ ] **EST over CoAP:** Consider migrating from static PSK to EST (Security Mode 4) for dynamic certificate provisioning.

#### 2. Data Formats
- [ ] **SenML-CBOR:** Migrate from TLV to SenML-CBOR for multi-resource payloads. ~30% smaller than TLV for typical sensor data.
- [ ] **LwM2M CBOR:** Evaluate for pure LwM2M deployments wanting minimum payload size.
- [ ] **Content-format negotiation:** Server must check client's `lwm2m` version before requesting new formats. v1.0 clients only support TLV and JSON.
- [ ] **Backward compatibility:** If mixed fleet (v1.0 + v1.2), server must negotiate format per-client.

#### 3. Operations
- [ ] **Composite Observe:** Replace multiple individual observations with Composite Observe to reduce observation count and notification traffic. At scale: 5 observations per device → 1 composite observation.
- [ ] **Send operation:** Evaluate for event-driven telemetry. Device pushes data without prior Read request. Reduces server-initiated traffic.
- [ ] **Bootstrap-Pack-Request:** If bootstrap performance matters, use `/bspack` for single round-trip provisioning (~25% traffic reduction).

#### 4. Objects
- [ ] **Object 21 (OSCORE):** If deploying OSCORE, provision security context.
- [ ] **Object 22 (MQTT Server):** If adding MQTT transport.
- [ ] **Object 23/24 (Gateway):** If deploying gateway architecture.
- [ ] **Object versioning:** Declare updated object versions in registration via `ver` attribute.

#### 5. Notification Attributes
- [ ] **con=true:** Use confirmable notifications for critical alerts (battery low, security events).
- [ ] **hqmax:** Enable historical notification buffering for time-series data during sleep.
- [ ] **edge:** Use for boolean threshold alerts (door open/close, tamper detect).
- [ ] **epmin/epmax:** Tune evaluation periods separately from notification periods.

#### 6. Backward Compatibility Rules
```
v1.2 client connecting to v1.0 server:
  ├── MUST use only v1.0 mandatory features
  ├── MUST NOT use Composite operations, Send, or new formats
  ├── CAN advertise lwm2m=1.2 in registration
  └── Server ignores unknown registration parameters

v1.0 client connecting to v1.2 server:
  ├── Server MUST NOT send Composite or Send operations
  ├── Server MUST negotiate TLV or JSON format
  ├── Server CAN manage client using v1.0 operations only
  └── Server reads lwm2m=1.0 from registration and adapts
```

---

## Specification Documents per Version

Each version produces a set of specification documents:

| Document Type | Abbreviation | Purpose |
|---------------|-------------|---------|
| Technical Specification (Core) | TS Core | Messaging, data model, interfaces, operations |
| Technical Specification (Transport) | TS Transport | Transport bindings, security configurations |
| Enabler Release Definition | ERELD | Feature list, scope, what changed |
| Enabler Test Specification | ETS | Conformance test cases, test architecture |
| Requirements Document | RD | Use cases and requirements (input to TS) |
| Architecture Document | AD | High-level architecture and design rationale |
| Supplemental Documents | SUP | Implementation guidelines, best practices |

### Document Naming Convention
```
OMA-{type}-LightweightM2M_{part}-V{major}_{minor}_{patch}-{date}-{status}
```
Example: `OMA-TS-LightweightM2M_Core-V1_2_2-20240613-A`
- `A` = Approved
- `C` = Candidate
- `D` = Draft

---

## Backward Compatibility Rules

1. A v1.2 client connecting to a v1.0 server must use only v1.0 mandatory features
2. The `lwm2m` registration parameter advertises the client's supported version
3. Content-format negotiation via CoAP Accept option ensures format compatibility
4. Object versioning (`ver` in registration) allows servers to understand client capabilities
5. New interfaces (Send, Composite operations) are only used when both sides support v1.2+
6. Security protocol negotiation (DTLS/TLS version, cipher suites) follows standard TLS/DTLS rules

---

## 3GPP Cross-References

LwM2M specifications reference these 3GPP documents:

| 3GPP Spec | Title | LwM2M Relevance |
|-----------|-------|-----------------|
| TS 23.401 | GPRS enhancements for E-UTRAN | CIoT architecture for LTE-M/NB-IoT transport |
| TS 23.682 | Architecture for MTC | Machine-Type Communication architecture |
| TS 24.008 | Mobile radio interface L3 (CS) | SMS delivery for SMS transport binding |
| TS 31.115 | Secured packet structure for (U)SIM | Smartcard bootstrap security |
| TS 36.133 | E-UTRA RRM requirements | Signal strength measurements for Object 4 |
| TS 44.018 | GSM/EDGE RRC protocol | 2G fallback connectivity monitoring |
| TS 23.122 | NAS for idle mode | Cell selection for connectivity objects |
