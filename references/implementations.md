# LwM2M Implementations — Open Source, Commercial & TestFest Product Listing

## Table of Contents
1. [OMA-Listed Open Source Implementations](#oma-listed-open-source-implementations)
2. [Additional Client Implementations](#additional-client-implementations)
3. [Server Implementations](#server-implementations)
4. [TestFest Product Listing](#testfest-product-listing)
5. [Testing & Conformance](#testing--conformance)
6. [Interoperability Notes](#interoperability-notes)
7. [Development Tools](#development-tools)

---

## OMA-Listed Open Source Implementations

The following are officially listed on openmobilealliance.org/specifications/resources/implementations.

### Eclipse Leshan (Java) — Server + Client
- **Repo:** github.com/eclipse-leshan/leshan | **License:** EDL + EPL-2.0
- **LwM2M:** v1.0, v1.1, v1.2 | **Transport:** CoAP/UDP (Californium), CoAP/TCP
- **DTLS:** Scandium — CID (ext type 54), PSK, RPK, x509
- **Demo:** leshan.eclipseprojects.io — public test sandbox
- **Strengths:** Reference Java implementation. Web UI. Cluster routing via CID node-ID prefix. Active community. Most-used server for TestFest.

### Leshan Sandbox — Hosted Test Environment
- Public, always-available LwM2M server for testing clients. Web UI shows registration, object tree, and observation management.

### Eclipse Wakaama / liblwm2m (C) — Client + Server
- **Repo:** github.com/eclipse-wakaama/wakaama | **License:** EDL + EPL-2.0
- **LwM2M:** v1.0, partial v1.1 | **Transport:** CoAP/UDP
- **DTLS:** External — mbedTLS, TinyDTLS, wolfSSL
- **Targets:** Linux, RTOS, bare-metal MCU (~20KB flash client core)
- **Limitations:** No TCP/TLS, no OSCORE, no v1.2 composite/send/gateway in mainline.

### Zephyr LwM2M Client — RTOS
- **Repo:** github.com/zephyrproject-rtos/zephyr (subsys/net/lib/lwm2m/)
- **License:** Apache 2.0 | **LwM2M:** v1.0, v1.1 (partial v1.2)
- **DTLS:** mbedTLS (nrf_security) or TinyDTLS
- **Targets:** nRF9160/nRF9161 (NB-IoT/LTE-M), nRF52, STM32, ESP32
- **NIDD:** Native support via LTE Link Controller on nRF91 targets.
- **Reference:** boaks/zephyr-coaps-client — CID-enabled client on nRF9160.

### Anjay (C) — AVSystem Client SDK
- **Repo:** github.com/AVSystem/Anjay | **License:** Apache 2.0 / Commercial
- **LwM2M:** v1.0, v1.1, v1.2 (most complete open-source v1.2 support)
- **Transport:** CoAP/UDP, TCP, SMS, Non-IP (NIDD), MQTT
- **DTLS:** mbedTLS, OSCORE, EST (Security Mode 4), HSM support
- **Targets:** Linux, Zephyr, FreeRTOS, bare-metal, Mbed OS
- **Strengths:** Full v1.2 — Composite ops, Send, LwM2M CBOR, Gateway. Excellent docs.
- **TestFest:** v1.0, v1.1, v1.2 compliance (SVE-41, Raleigh June 2024).

### IOWA (C) — IoTerop Client + Server SDK
- **Repo:** github.com/IOTEROP/IOWA (evaluation SDK)
- **License:** Commercial (eval freely available)
- **LwM2M:** v1.0, v1.1, v1.2 | **Claim:** Most compact C implementation.
- **Targets:** Cortex-M0+ to Cortex-M33, any RTOS, Linux
- **NIDD/LPWAN:** NB-IoT, LTE-M, LoRaWAN native support.
- **TestFest:** Client + server tested 2016–2024 (v1.0, v1.1, v1.2).

### Friendly LwM2M Client — Friendly Technologies
- **Repo:** github.com/Friendly-Technologies/Friendly-LwM2M-Client
- **Language:** C/C++ (CMake) | **License:** Open Source
- **LwM2M:** v1.0, v1.1 (v1.2 in development)
- **Transport:** CoAP/UDP, NIDD | **DTLS:** mbedTLS, TinyDTLS, wolfSSL
- **Targets:** Linux, OpenWRT, Raspberry Pi 4, RPi4, ARM32/64, nRF9160, nRF9161, Zephyr OS, FreeTOS future
- **Based on:** Wakaama/liblwm2m core, many new additions, fixed DTLS issues, abstraction and platform HAL.
- **Ecosystem:** Part of Friendly One-IoT™ DMP — the earliest commercial LwM2M deployments (2017). Server supports v1.0/1.1/1.2/1.2.1 plus OMA-DM, TR-069, USP, MQTT, CoAP/S. Fortune 350+ telco deployments scaling to 1B+ NB-IoT devices.

---

## Additional Client Implementations

### Awa LightweightM2M — Imagination Technologies
- **Repo:** github.com/FlowM2M/AwaLWM2M | **License:** BSD-3 | **LwM2M:** v1.0
- Full client+server with intuitive API. Community maintained.

### ARM mbed Client (C++) — Legacy
- **LwM2M:** v1.0 | **DTLS:** mbedTLS | **Status:** Maintenance mode.
- mbed Cloud (now Pelion/Izuma) was historically influential. TestFest 2016–2018.

### Nordic nRF Connect SDK (via Zephyr)
- Nordic's nRF Connect SDK bundles Zephyr LwM2M with nrf_security (CryptoCell HW acceleration), LTE Link Controller (NB-IoT/LTE-M), and modem AT interface for NIDD.

---

## Server Implementations

### Eclipse Californium (Java) — CoAP/DTLS Engine
- **Repo:** github.com/eclipse/californium | **DTLS:** Scandium
- CID (ext type 54), cluster routing, graceful restart, session persistence.

### Coiote IoT DM — AVSystem (Commercial)
- v1.0/1.1/1.2. Full lifecycle DM, FOTA orchestration, multi-tenancy, REST API.
- Most prolific TestFest participant (2016–2024).

### Nokia IMPACT (Commercial)
- Standards-based IoT platform. TestFest Seoul 2018/2019 (v1.0, v1.1 server).

### Motive (Commercial)
- Standards-based IoT platform. v1.0, v1.1 server.

### Ericsson Composition Engine (Commercial)
- Service exposure + LwM2M DM. TestFest 2016–2019 (v1.0, v1.1 server).

### Huawei OceanConnect (Commercial)
- Unified IoT platform with LwM2M. TestFest Pittsburgh 2017 (v1.0 server).

### Cumulocity IoT — Software AG (Commercial)
- Cloud IoT platform with LwM2M module, based on Leshan OSS Server. TestFest SVE-40 2023 (v1.0, v1.1 server).

### Friendly One-IoT™ DMP — Friendly Technologies (Commercial)
- v1.0/1.1/1.2/1.2.1 server. Multi-protocol (LwM2M, OMA-DM, TR-069, USP, MQTT). Cloud-agnostic (Azure, AWS, GCP). Multi-tenancy. TestFest Warsaw 2017.

### InterDigital IoT Platform
- oneM2M-rooted platform with integrated LwM2M client+server. TestFest Pittsburgh 2017.

### ThingsBoard (Open Source)
- IoT platform with LwM2M via Californium/Leshan. Dashboard, rule engine.

### FIWARE (Open Source)
- Smart Cities Data platform with limited LwM2M capability.


---

## TestFest Product Listing

Source: openmobilealliance.org/specifications/resources/product-listing

### TestFest Event History

| Event | Location | Date | Versions |
|-------|----------|------|----------|
| TF-32 | San Diego, USA | Jan 2016 | v1.0 |
| TF-33 | Singapore | Oct 2016 | v1.0 |
| TF-34 | Pittsburgh, USA | May 2017 | v1.0 |
| TF-35 | Warsaw, Poland | Oct/Nov 2017 | v1.0 |
| TF-36 | Seoul, Korea | Jul 2018 | v1.0 |
| TF-37 | Seoul, Korea | Oct 2019 | v1.0, v1.1 |
| VTF-38 | Virtual | Apr 2020 | v1.1 |
| VTF-39 | Virtual | Mar 2021 | v1.1 |
| SVE-40 | Virtual | Nov 2023 | v1.0, v1.1 |
| SVE-41 | Raleigh, NC USA | Jun 2024 | v1.0, v1.1, v1.2 |

### Tested Clients

| Company | Product | Versions | Latest Event |
|---------|---------|----------|--------------|
| AVSystem | Anjay SDK | v1.0, v1.1, v1.2 | SVE-41 (2024) |
| IoTerop | IOWA SDK | v1.0, v1.1 | SVE-41 (2024) |
| Aetheros | AOS Smart Metering | v1.0, v1.1 | SVE-40 (2023) |
| Bulk Tainer Telematics | Tank Tracker | v1.1 | SVE-40 (2023) |
| Altair/Sony | ALT1250 NB-IoT | v1.0, v1.1 | TF-37 (2019) |
| III (Taiwan) | III DM Solution | v1.0, v1.1 | TF-37 (2019) |
| Friendly Technologies | One-IoT Client | v1.0 | TF-35 (2017) |
| ARM | mbed Client | v1.0 | TF-36 (2018) |
| AhnLab | Security Client | v1.0 | TF-36 (2018) |
| Gemalto (Thales) | IoT Module | v1.0 | TF-36 (2018) |
| Hancom MDS | NeoIDM Client | v1.0 | TF-36 (2018) |
| Kona S | KonaS DM | v1.0 | TF-36 (2018) |
| Paradox Engineering | PE Smart Urban Network | v1.1 | VTF-39 (2021) |
| Smith Micro | QuickLink IoT | v1.0 | TF-35 (2017) |
| Telit | IoT Module/Platform | v1.0 | TF-35 (2017) |
| Altiux | BoxPwr Middleware | v1.0 | TF-32 (2016) |
| Centero | FabriK Thread | v1.0 | TF-34 (2017) |
| ETRI | Research Client | v1.0 | TF-35 (2017) |

### Tested Servers

| Company | Product | Versions | Latest Event |
|---------|---------|----------|--------------|
| IoTerop | IOWA Server | v1.0, v1.1, v1.2 | SVE-41 (2024) |
| AVSystem | Coiote IoT DM | v1.0, v1.1 | VTF-39 (2021) |
| Aetheros | AOS Server | v1.0, v1.1 | SVE-40 (2023) |
| Cumulocity | IoT Platform | v1.0, v1.1 | SVE-40 (2023) |
| Nokia | IMPACT | v1.0, v1.1 | TF-37 (2019) |
| Ericsson | Composition Engine | v1.0, v1.1 | TF-37 (2019) |
| Friendly Technologies | One-IoT DMP | v1.0 | TF-35 (2017) |
| ARM | mbed Cloud | v1.0 | TF-36 (2018) |
| Huawei | OceanConnect | v1.0 | TF-34 (2017) |
| InterDigital | IoT Platform | v1.0 | TF-34 (2017) |
| III (Taiwan) | III DM Server | v1.0, v1.1 | TF-37 (2019) |
| Hancom MDS | NeoIDM Server | v1.0 | TF-36 (2018) |
| Kookmin University | Research Server | v1.0 | TF-36 (2018) |

### Notable Deployments
- **Friendly Technologies:** 1B+ NB-IoT device target. First commercial LwM2M smart water meter on NB-IoT (2017). Cloud-agnostic multi-tenancy.
- **Altair/Sony ALT1250:** NB-IoT chipset with LwM2M client deployed on AT&T, Verizon, China Mobile, KDDI, SoftBank, Vodafone.
- **Paradox Engineering:** Smart city deployments with uCIFI + IPSO Smart Objects (MinebeaMitsumi Group).
- **Aetheros AOS:** Utility-grade smart metering software.

---

## Testing & Conformance

### Test Specification Structure
- **ETS:** Enabler Test Specification — test architecture, test purposes, prerequisites
- **TP:** Test Purposes — objectives, preconditions, expected results
- **TE:** Test Execution — step-by-step procedures
- **TS:** Test Suite — grouped test cases by functionality
- **SCR:** Static Conformance Requirements (Appendix B of Core TS)

### Public Test Servers
| Server | URL | Features |
|--------|-----|----------|
| Leshan Demo | leshan.eclipseprojects.io | Web UI, v1.0/1.1/1.2 |
| Coiote Demo | eu.iot.avsystem.cloud | Full v1.2, cloud-hosted |
| Friendly One-IoT DMP | demodm1.friendly-tech.com/PortfolioPortal/iot | Full v1.2, cloud-hosted |

---

## Interoperability Notes

**TinyDTLS ↔ Californium:** CID extension type mismatch (53 draft vs 54 final RFC 9146). Silent negotiation failure.

**Wakaama ↔ Leshan:** Good v1.0 interop. SenML edge cases. Composite ops need patches.

**Content-Format:** v1.0 clients may ignore Accept option for SenML. Handle 4.06 gracefully.

**DTLS Resumption:** CID (ext type 54) is preferred over session ID/ticket resumption for v1.2+.

---

## Development Tools

- **OMA LwM2M Editor:** Web tool for creating Object XML files. OMNA registry submission.
- **OMNA Registry:** GitHub: OpenMobileAlliance/lwm2m-registry
- **Wireshark/tshark:** CoAP + DTLS dissectors. CID field support. See protocol-details.md.
- **libcoap:** C CoAP library with CLI tools (`coap-client`, `coap-server`)
- **aiocoap:** Python asyncio CoAP library — excellent for scripted testing
- **Copper (Cu4):** Firefox CoAP extension (Cu4 maintained fork)
