# OMA LwM2M Specification Versions — Detailed Reference

## Table of Contents
1. [Version Timeline](#version-timeline)
2. [v1.0 — Initial Release](#v10--initial-release-february-2017)
3. [v1.0.1 / v1.0.2 — Errata](#v101--v102--errata-july-2017--february-2018)
4. [v1.1 — Non-IP, TCP and Composite Operations](#v11--non-ip-tcp-and-composite-operations-july-2018)
5. [v1.1.1 — CBOR Enhancement](#v111--cbor-enhancement-june-2019)
6. [v1.2 — MQTT, HTTP, Gateway and LwM2M CBOR](#v12--mqtt-http-gateway-and-lwm2m-cbor-november-2020)
7. [v1.2.1 — Clarifications](#v121--clarifications-december-2022)
8. [v1.2.2 — Bug Fixes](#v122--bug-fixes-june-2024)
9. [v2.0 — In Development](#v20--in-development-non-normative)
10. [Spec Library Inventory](#spec-library-inventory)
11. [Specification Documents per Version](#specification-documents-per-version)
12. [Backward Compatibility Rules](#backward-compatibility-rules)
13. [3GPP Cross-References](#3gpp-cross-references)

---

## Version Timeline

Approved (`-A`) dates from the OMA release directories and Core TS 1.2.2 Table A.1-1.

| Version | Approved | Status | Key Theme |
|---------|----------|--------|-----------|
| v1.0 | 2017-02-08 | Approved | Foundational protocol (single TS) |
| v1.0.1 | 2017-07-04 | Approved | Errata |
| v1.0.2 | 2018-02-09 | Approved | Errata and clarifications |
| v1.1 | 2018-07-10 (Candidate 2018-06-12) | Approved | TCP/TLS, Non-IP, OSCORE, SenML, Composite ops, Send, EST |
| v1.1.1 | 2019-06-17 | Approved | CBOR single-value content format (60) |
| v1.2 | 2020-11-10 | Approved | MQTT, HTTP, Gateway, LwM2M CBOR, Profile ID, DTLS CID |
| v1.2.1 | 2022-12-09 | Approved | Clarifications, WebSockets binding |
| v1.2.2 | 2024-06-13 | Approved | Editorial and bug fixes (ERELD §3.5) |
| v2.0 | In development — v0.9 Candidate targeted Dec 2026 (non-normative, unverified) | Not published | Roadmap only; see §v2.0 |

---

## v1.0 — Initial Release (February 2017)

The foundational specification establishing the LwM2M protocol architecture. Approved 2017-02-08.

### Features
- **Four interfaces:** Bootstrap, Registration, Device Management & Service Enablement, Information Reporting
- **Transport bindings:** CoAP over UDP (with/without DTLS), CoAP over SMS (with/without DTLS)
- **Queue Mode:** Server queues operations while client is sleeping; client polls on registration update. In v1.0 syntax Queue Mode is folded into the binding value (`b=U`, `UQ`, `S`, `SQ`, `US`, `UQS`); from v1.1 it is the separate `Q` registration parameter and `b` carries only transport letters
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
- OMA-ERELD-LightweightM2M-V1_0, OMA-RD-LightweightM2M-V1_0, OMA-AD-LightweightM2M-V1_0 (the only LwM2M release with an Architecture Document)
- OMA-SUP-XML_LWM2M_{Object}-V1_0 — one machine-readable object definition per core object

---

## v1.0.1 / v1.0.2 — Errata (July 2017 / February 2018)

- **v1.0.1** (approved 2017-07-04): errata to the TS and to the Object 2, 3, 4, 5 and 7 XML definitions (`OMA-SUP-XML_*-V1_0_1`). No new features.
- **v1.0.2** (approved 2018-02-09): further bug fixes and editorial corrections. No new features.

---

## v1.1 — Non-IP, TCP and Composite Operations (July 2018)

Major expansion of transport bindings, security options and the operation set. Candidate 2018-06-12, approved 2018-07-10.

### New Features
- **CoAP over TCP/TLS (RFC 8323):** NAT-friendly persistent connections; TLS 1.2 (RFC 5246) for the TCP binding. Binding letter `T`
- **Non-IP transports** (binding letter `N`):
  - 3GPP CIoT: NB-IoT Data-over-NAS (DoNAS) and User Plane CIoT optimisation
  - LoRaWAN: CoAP mapping over LoRaWAN; NoSec at the LwM2M layer, security provided by the LoRaWAN link (Transport TS 1.2.2 §6.8.4)
- **OSCORE (RFC 8613) and Object 21 (LwM2M OSCORE):** Application-layer CoAP security independent of transport. Survives proxies. Object 21 is the only core-TS object added in v1.1
- **SenML data formats:**
  - SenML-JSON (Content-Format 110): JSON-based, human-readable
  - SenML-CBOR (Content-Format 112): Binary, highly efficient for constrained devices
- **Composite operations** (Core TS 1.1 §6.3.8–6.3.9, §6.4.4): Read-Composite, Write-Composite, Observe-Composite and Cancel Observation-Composite — multiple paths across objects in a single request, SenML payloads
- **Send operation** (Core TS 1.1 §6.3.10; relocated to Information Reporting §6.4.6 in v1.2): client-initiated data push without a prior Read; controlled by Mute Send /1/x/23
- **EST — Security Mode 4** (Transport TS 1.1 "Certificate mode with EST"): Enrollment over Secure Transport for certificate provisioning
- **Notification attributes epmin/epmax:** Minimum/Maximum Evaluation Period
- **Registration parameter changes:** `ep` becomes optional (Table 6.2.1-1: "Endpoint Client Name — Required: No"); Queue Mode becomes the separate `Q` parameter; binding `b` carries transport letters only
- **Security Object (0) Resources 13–17:** Matching Type, SNI, Certificate Usage, Ciphersuite, OSCORE Security Mode. The SNI Resource /0/x/14 appears here
- **Resource Instance level access:** Addressing individual resource instances within multi-instance resources
- **Enhanced bootstrapping:** Incremental bootstrap updates, improved PKI support
- **Extended registration:** Enhanced registration sequence mechanisms (Initial Registration Sequence)
- **New data types:** Unsigned Integer, Corelnk
- **Performance:** Multi-object read/write optimisations via SenML

### Specification Structure Change
Starting with v1.1, the specification is split into two documents:
- **Core TS** — messaging layer, data model, interfaces, operations
- **Transport TS** — transport-specific mappings, security configurations

### Objects from companion enablers (not part of the LwM2M ER)
These objects are often listed as "v1.1 objects" but are defined by separate OMA enablers/reference releases, each with its own TS or XML and release date. The only core-TS object added in v1.1 is Object 21 (OSCORE).

| Object | Name | Defining enabler | Release | Notes |
|--------|------|------------------|---------|-------|
| 8 | LwM2M Lock and Wipe | LWM2M_LOCKWIPE RR | 1.0.4, 2024-03-12 | |
| 9 | LwM2M Software Management | LWM2M_SWMGMT RR | 1.0.3, 2024-03-12 | |
| 14 | LwM2M Software Component | LWM2M_SWMGMT RR | 1.0.3, 2024-03-12 | Companion to Object 9 |
| 10 | Cellular Connectivity | LWM2M_CONNMGMT RR 1.0 (2017-03-14, now Historic) | continued as ObjLwM2M_Cell_Conn 1.1, 2020-11-10 | |
| 11 | APN Connection Profile | LWM2M_CONNMGMT RR 1.0 (Historic) | ObjLwM2M_APN_Conn 1.1, 2020-11-10 | |
| 12 | WLAN Connectivity | LWM2M_CONNMGMT RR 1.0 (Historic) | ObjLwM2M_WLAN_Conn 1.1.1, 2022-10-03 | |
| 13 | Bearer Selection | LWM2M_CONNMGMT RR 1.0 (Historic) | ObjLwM2M_Bearer_Conn 1.1, 2020-11-10 | |
| 15 | Device Capability Management | LWM2M_DevCapMgmt RR | 2.0.1, 2024-03-12 | Object v2.0 |
| 16 | Portfolio | LWM2M_PortfolioObj RR | 1.0.1, 2020-06-16 | |
| 18 | Non-Access Stratum (NAS) Configuration | OMNA registry entry only | — | Not in the OMA spec library (unverified against OMA spec library) |
| 19 | BinaryAppDataContainer | LwM2M_APPDATA ER | 1.0 2018-06-15; latest 1.0.2, 2024-03-12 | |
| 20 | Event Log | LwM2M_EventLog ER | 1.0 2018-06-15; latest 3.0, 2025-07-14 | Object v3.0 |
| 22 | Virtual Observe Notify | LwM2M_VON ER | 1.0, 2019-07-11 (+ Guidelines 1.1 Draft, 2021-06-21) | Object v1.1 |

---

## v1.1.1 — CBOR Enhancement (June 2019)

Approved 2019-06-17.

### New Features
- **CBOR for single resources (Content-Format 60):** Allows CBOR encoding for Read and Write operations on individual resources. Previously CBOR was only available via SenML-CBOR.

---

## v1.2 — MQTT, HTTP, Gateway and LwM2M CBOR (November 2020)

The largest feature release since v1.0. Approved 2020-11-10. Feature list per ERELD 1.2 §3.4 and Core/Transport TS 1.2.

### New Features

**Transports:**
- **MQTT transport binding** (Transport TS §8): LwM2M messaging over MQTT pub/sub; topic structure §8.2, CBOR mapping §8.7, configuration via new MQTT Server Object (24). Binding letter `M`
- **HTTP transport binding** (Transport TS §7): LwM2M over HTTP/HTTPS. Binding letter `H`

**Registration and Bootstrap:**
- **Profile ID (`pid`)** (Core TS §6.2.1, Table 6.2.1-1): the Object/Object-Instance list by reference rather than by value. ASCII string, ABNF `preConfigID = ("oma"/"v") ":" ...` or `dynamicID = suite ":" hex`; multiple `pid` MAY be sent and mixed with an explicit list; Objects 0, 21, 23 MUST NOT be included; server MAY refuse with an error, after which the client re-registers with the full list. Server support is optional
- **Bootstrap-Pack-Request** (Transport TS §6.4.2): CoAP **GET** `/bspack?ep={name}` with an Accept option (SenML CBOR / SenML JSON / LwM2M CBOR); the whole bootstrap configuration is returned in one response. (`acc` query parameter added in v1.2.1)
- Registration and bootstrap optimisations reduce bytes on the wire (ERELD 1.2)

**Data Formats:**
- **LwM2M CBOR** (Content-Format 11544): A CBOR encoding optimised for the LwM2M data model. More compact than SenML-CBOR for LwM2M-specific payloads

**Gateway:**
- **LwM2M Gateway support:** Non-LwM2M devices and LwM2M devices behind a gateway are managed through the gateway, which is the sole registered endpoint; Objects 25 (LwM2M Gateway, v1.0) and 26 (LwM2M Gateway Routing) are defined in Core TS 1.2 Appendix E. Object 25 v2.0 is defined later by the separate LwM2M Gateway enabler (TS 1.1 / 1.1.1)

**Notification Attributes (new):**
- `edge` — Trigger notification on rising/falling edge (`edge=0|1`)
- `con` — Confirmable notifications (`con=0|1`)
- `hqmax` — Maximum Historical Queue depth for time-series buffering
- Observation attributes may now be carried in the Observe request itself (ERELD 1.2)

**Security:**
- **DTLS CID (RFC 9146):** Connection ID support for DTLS 1.2. Critical for sleepy-device NAT traversal
- **TLS 1.3 (RFC 8446) and DTLS 1.3 (RFC 9147):** optional ("This version of the specification MAY optionally use TLS 1.3 and/or DTLS 1.3", Transport TS §5.2.1); controllable through Security Object configuration
- **SNI (RFC 6066):** the v1.2 text carried a blanket MUST for the SNI extension; v1.2.1 narrowed it to deployments without DNS (Transport TS 1.2.2 §5.2.9.6.1)
- "Untangling" of security credentials from server configuration (ERELD 1.2)

**Operations:**
- Send relocated to the Information Reporting interface (Core TS §6.4.6); Composite operations and Send themselves date from v1.1

**Firmware Update:**
- Enhanced functionality for firmware updates (ERELD 1.2); Object 5 revised. Object 5 remains single-instance — multi-component software is handled by Objects 9/14

**Other:**
- Clarified object versioning rules (Core TS §7.2)
- Updated Endpoint Client Name URN format recommendations (§7.4.1)

### New and modified Objects
- **New:** 23 (LwM2M COSE), 24 (MQTT Server), 25 (LwM2M Gateway, v1.0), 26 (LwM2M Gateway Routing), 27 (5G NR Connectivity)
- **Modified:** 0 (Security, v1.2 — R18–30), 1 (Server, v1.2), 5 (Firmware Update — Core TS 1.2.2 defines `urn:oma:lwm2m:oma:5:1.2`), 21 (OSCORE)
- Object 22 (Virtual Observe Notify) is **not** a v1.2 deliverable — it belongs to the separate VON enabler (2019-07-11)

---

## v1.2.1 — Clarifications (December 2022)

Approved 2022-12-09. Changes per Core TS 1.2.2 §1.2 and Transport TS 1.2.2 §1.2 ("Version 1.2.1"):

### Core
- Added the optional `acc` parameter to Bootstrap-Pack-Request
- Fixed a potential ambiguity in the LwM2M CBOR encoding of the Create operation
- Rewrote the notification attribute text for clarity and simplification
- Clarified that Execute operation arguments must be unique
- Clarified that the LwM2M COSE Object (23) must not be part of the Register "Objects and Object Instances" parameter (COSE and MQTT Server objects in Bootstrap-Discover responses also clarified)
- LwM2M CBOR content format number (11544) registered with IANA
- Consolidated the recommended URN formats for the Endpoint Client Name
- Moved the Attributes section into the Identifiers and Resources section

### Transport
- Clarified that the CoAP binding can be used over WebSockets as defined in RFC 8323 (new §6.8.6 WebSockets Binding; `coap+ws://` port 80, `coaps+ws://` port 443)
- Removed the ambiguous requirement to re-register when a new security context is created in the CoAP binding
- Clarified multiple X.509 certificates in the "Public Key or Identity" Resource
- Added Send to the exceptions to Confirmable CoAP message usage
- Clarified Alternate Path usage in SenML encodings; added the missing SenML JSON/CBOR content formats to Read-Composite; added "4.01 Unauthorized" to the Register CoAP binding; fixed the CIoT binding reference
- SNI: blanket MUST dropped; SNI extension support is MUST only for deployments without DNS (§5.2.9.6.1)

Backward compatible with v1.2.

---

## v1.2.2 — Bug Fixes (June 2024)

Approved 2024-06-13. Editorial/bug-fix release; list per ERELD 1.2.2 §3.5:

- Updated the RFC 2119 reference to RFC 8174
- Fixed broken links to appendix references
- Clarification of core object versioning
- Fixed the Object 27 range enumeration
- Transport TS Appendix C (CoAP over LoRaWAN) diagrams updated
- Clarified UDP bindings in Transport TS §6.2
- Clarified the CoAP error codes in §6.4.4
- Updated the ETS test for MQTT

Backward compatible with v1.2 and v1.2.1. This is the latest approved release; the Core/Transport TS 1.2.2 are the normative baseline for everything in this skill.

---

## v2.0 — In Development (non-normative)

> **Non-normative — roadmap as of 2026-10-10.** Not defined in any published OMA specification (latest approved: LwM2M 1.2.2, 2024-06-13). Treat as indicative. Nothing in this section can be verified against the OMA spec library; cite it as roadmap, never as spec.

Under development by the DMSO IoT Working Group at OMA SpecWorks. Reported from the OMA Plenary (Kraków, April 2026): triage of ~70 issues complete and work organised on GitHub with team leads per technical area (unverified).

### Reported work items (unverified)

- **Profile IDs:** the Profile ID (`pid`) registration parameter already exists in v1.2 (Core TS 1.2.2 §6.2.1 — ASCII `oma:`/`v:`/`suite:hex` strings, not a 4-byte integer). v2.0 work may extend it with pre-registered vertical profiles (smart meters, trackers, sensors, gateways) — unverified.
- **Simplified Bootstrapping:** further bootstrap-traffic reduction for caDM (constrained Application DM), B-IoT (Broadband IoT) and RedCap devices (vendor/roadmap estimate, unverified; no quantified saving is sourced).
- **Advanced Firmware Update Enabler:** push/pull delivery and application-layer updates with independently managed app layers; delta (differential) updates (vendor/roadmap estimate, unverified; no quantified saving is sourced).
- **eSIM Remote Provisioning:** Objects 504 (Remote SIM Provisioning, OMA) and 3443 (eSIM IoT, GSMA, `ext` range) are **already-registered OMNA objects**, not v2.0 deliverables (registry-mirror evidence; verify against OMNA). v2.0 work may reference them together with the GSMA SGP.32 bootstrap flow.
- **Edge Computing Proxy Support:** edge proxies that cache, aggregate and filter LwM2M messages; QoS-aware scheduling.
- **DTLS 1.3:** already optional in v1.2 (Transport TS 1.2.2 §5.2.1). Any v2.0 change would be to its requirement level — unverified.
- **QoS-aware Services:** differentiated service levels per observation.
- **Digital Twinning:** Smart Cities WG activity in parallel with DMSO v2.0 work.

### Northbound API (companion specification — non-normative, unverified status)
A companion specification reported to be in development in a separate WG:
- **Purpose:** a common API layer above any compliant LwM2M server for back-end systems (OSS/BSS, cloud platforms, enterprise IT).
- **Problem addressed:** back-end systems today integrate uniquely with each LwM2M server vendor.
- **Status:** reported "nearing feature-complete" (April 2026) — unverified; no OMA document is published.
- **Prior art in the library:** OMA-RD-M2MInterface-V1_0 (2015-03-24) already defines requirements for a DM-server northbound interface (see Spec Library Inventory).

### Timeline (single, unverified)
- v0.9 Candidate targeted December 2026 (unverified)
- Thereafter: ETS updates, conformance/validation roadmap, full approval — no dates sourced

---

## Spec Library Inventory

Snapshot of the OMA LwM2M document library this skill was reconciled against (2026-10-10). "Latest" is the newest release directory present; status letters per the OMA README (A Approved, C Candidate, H Historic; D Draft is a document state, not a release state).

| Enabler | Type | Latest | Date | Status | Defines |
|---------|------|--------|------|--------|---------|
| LightweightM2M | ER | 1.2.2 | 2024-06-13 | A | Core TS + Transport TS, ERELD, RD 1.2, SUP-XML XSD, ERP |
| LightweightM2M ETS | ETS (`_INT`) | 1.2.1 | 2024-03-12 | C | Interoperability test cases only (ETS 1.2 INT A 2023-10-03) |
| LwM2M Gateway | ER | 1.1.1 | 2024-03-12 | A | Object 25 v2.0 (TS + SUP-XML_25) |
| LwM2M Virtual Observe Notify (VON) | ER | 1.0 | 2019-07-11 | A | Object 22; plus Guidelines 1.1, 2021-06-21, D |
| LwM2M BinaryAppDataContainer | ER | 1.0.2 | 2024-03-12 | A | Object 19 |
| LwM2M EventLog | ER | 3.0 | 2025-07-14 | A | Object 20 v3.0 (1.0 → 2.0 multi-instance → 2.1 → 3.0 Syslog/CLF) |
| LWM2M DevCapMgmt | RR | 2.0.1 | 2024-03-12 | A | Object 15 v2.0 |
| LWM2M LockWipe | RR | 1.0.4 | 2024-03-12 | A | Object 8 |
| LWM2M SwMgmt | RR | 1.0.3 | 2024-03-12 | A | Objects 9 and 14 |
| LWM2M Portfolio | RR | 1.0.1 | 2020-06-16 | A | Object 16 |
| LWM2M ConnMgmt | RR | 1.0 | 2017-03-14 | H | Objects 10–13 (Historic; continued as standalone object releases) |
| M2M Interface | RR | 1.0 | 2015-03-24 | A | RD for a DM-server northbound interface (prior-art NBI requirements) |
| M2M Device Classification | RR | 1.0 | 2012-10-30 | A | White Paper: seven-attribute device classification |
| Standalone object releases `ObjLwM2M_*` | SUP-XML | per object | 2019–2022 | A | 0 v1.2, 1 v1.2, 2 v1.1, 3 v1.2, 4 v1.3.1, 5 v1.1, 6 v1.0.3, 7 v1.0.5, 10/11/13 v1.1, 12 v1.1.1, 21 v2.0 (2021-11-23), 23/24/25(v1.0)/26/27 v1.0 |

### ETS lineage
OMA-ETS-LightweightM2M: 1.0 (Candidate/Draft iterations 2014–2017), 1.0.1 A 2017-09-26, 1.0.2 A 2018-08-15, 1.1 D only (2019-09-12, never approved); then renamed **OMA-ETS-LightweightM2M_INT**: 1.2 A 2023-10-03, 1.2.1 C 2024-03-12. The `_INT` ETS contains interoperability test cases only — its §5 "LightweightM2M Conformance Test Cases" reads "None." — and Core TS Appendix B (Static Conformance Requirements) is voided. There is no OMA conformance certification derived from the ETS. The 1.2.x ETS includes `int-110`/`int-111` (Profile ID registration). The EVP (Enabler Validation Plan, 1.0 2014) series is discontinued; its content moved into the ETS appendix.

### Legacy enablers
- **M2M Interface RD 1.0** (OMA-RD-M2MInterface-V1_0-20150324-A): requirements for an interface from a DM server to the M2M service layer — the northbound interface (NBI). Prior art for any LwM2M NB API discussion; requirements only, no protocol.
- **M2M Device Classification WP** (OMA-WP-M2M_Device_Classification-20121030-A): classifies M2M devices by seven physical/logical attributes, written `C(attr1,…,attr7)` (e.g. `C(WW, WL, NFW, FL, NIP, NIO, PS)`). An OMA/ETSI-M2M framework — not the RFC 7228 constrained-node classes (C0/C1/C2).

---

## Specification Documents per Version

Document types used in the LwM2M library (per the OMA release README):

| Abbreviation | Document Type | LwM2M usage |
|--------------|---------------|-------------|
| TS | Technical Specification | Core and Transport TS (from v1.1); single TS in v1.0.x; companion-enabler TSs |
| ER / ERELD / ERP | Enabler Release / Enabler Release Definition / Enabler Release Package | ERELD lists scope, feature deltas and the document set; ERP is the zip of the release |
| RR / RRELD / RRP | Reference Release / Definition / Package | Used by the companion object releases (LockWipe, SwMgmt, DevCapMgmt, Portfolio, ConnMgmt) and legacy M2M enablers |
| ETS | Enabler Test Specification | `OMA-ETS-LightweightM2M_INT` — interoperability test cases only |
| ETR | Enabler Test Requirements | Not present in the LwM2M library |
| EVP | Enabler Validation Plan | Discontinued (1.0, 2014); content folded into the ETS appendix |
| RD | Requirements Document | Use cases and requirements (input to TS); RD 1.0, 1.1, 1.2 |
| AD | Architecture Document | v1.0 only (OMA-AD-LightweightM2M-V1_0) |
| SUP | Support Document | Machine-readable object XML (`OMA-SUP-XML_{ObjID}`) and the LwM2M XSD schema |
| DDF / DDS / DTD / XSD / XML | Device Description Framework / Data Definition Spec / DTD / Schema / XML | Object definition formats; the LwM2M object DDF is the SUP-XML |
| WP | White Paper | M2M Device Classification |
| AC / API / ASN / MO / OD / TFP / WIDL / WSDL | Application Characteristics / API / ASN.1 / Management Object / Overview / Test Files Package / interface definitions | Listed by OMA; not used by the LwM2M releases |

### Document Naming Convention
```
OMA-{type}-LightweightM2M_{part}-V{major}_{minor}_{patch}-{YYYYMMDD}-{status}   # Core / Transport TS
OMA-ERELD-LightweightM2M-V{x}_{y}_{z}-{YYYYMMDD}-{status}                       # ERELD has no part
OMA-ETS-LightweightM2M_INT-V{x}_{y}_{z}-{YYYYMMDD}-{status}                     # _INT ETS (1.2+)
OMA-TS-LWM2M_{Enabler}-V{x}_{y}_{z}-{YYYYMMDD}-{status}                         # companion enablers
OMA-SUP-XML_{ObjID}-V{x}_{y}[_{z}]-{YYYYMMDD}-{status}.xml                      # object definitions
V{x}_{y}[_{z}]-{YYYYMMDD}-{status}/                                             # release directory
```
Examples: `OMA-TS-LightweightM2M_Core-V1_2_2-20240613-A`, `OMA-SUP-XML_25-V2_0-20240312-A.xml`, `V1_2_2-20240613-A/`.
- `A` = Approved (after public comment and any interop testing)
- `C` = Candidate (approved by OMA, published for comment)
- `H` = Historic (obsolete, no longer maintained — e.g. ConnMgmt 1.0)
- `D` = Draft (working document; not a release state)

---

## Backward Compatibility Rules

1. A v1.2 client connecting to a v1.0 server must use only v1.0 mandatory features
2. The `lwm2m` registration parameter advertises the client's supported version (mandatory; MUST match the approved version number of the TS it implements)
3. Content-format negotiation via CoAP Accept option ensures format compatibility
4. Object versioning (`ver` in registration) allows servers to understand client capabilities
5. Send and the Composite operations require both peers to be ≥ v1.1 (they were introduced in v1.1, not v1.2)
6. Security protocol negotiation (DTLS/TLS version, cipher suites) follows standard TLS/DTLS rules

---

## 3GPP Cross-References

LwM2M specifications reference these 3GPP documents:

| 3GPP Spec | Title | LwM2M Relevance |
|-----------|-------|-----------------|
| TS 23.401 | GPRS enhancements for E-UTRAN | CIoT architecture for LTE-M/NB-IoT transport |
| TS 23.682 | Architecture for MTC | Machine-Type Communication architecture |
| TS 23.720 | Architecture enhancements for CIoT | CIoT binding (Transport TS 1.2.2 §6.8.5) |
| TS 24.008 | Mobile radio interface L3 (CS) | SMS delivery for SMS transport binding |
| TS 31.115 | Secured packet structure for (U)SIM | SMS Secured Packet mode and smartcard bootstrap security |
| TS 36.133 | E-UTRA RRM requirements | Signal strength measurements for Object 4 |
| TS 44.018 | GSM/EDGE RRC protocol | 2G fallback connectivity monitoring |
| TS 23.122 | NAS for idle mode | Cell selection for connectivity objects |
