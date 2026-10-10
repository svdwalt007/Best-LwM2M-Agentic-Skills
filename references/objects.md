# OMA LwM2M Objects — Reference

## Table of Contents
1. [Object Model Overview](#object-model-overview)
2. [Core Objects (0–7)](#core-objects-07)
3. [Extended OMA object resource maps](#extended-oma-object-resource-maps)
4. [Extended OMA Objects (8–28)](#extended-oma-objects-828)
5. [IPSO Smart Objects](#ipso-smart-objects)
6. [Industry-Registered Objects](#industry-registered-objects)
7. [Object ID Allocation Ranges](#object-id-allocation-ranges)
8. [Object Versioning](#object-versioning)
9. [Reusable Resources](#reusable-resources)
10. [OMNA Registry Process](#omna-registry-process)

Source conventions used below: resource tables are transcribed from the OMA object definition files (`OMA-SUP-XML_<id>`, cited as "OMA-SUP-XML_N vX.Y") and, where the Core TS carries a newer copy, from Core TS 1.2.2 Appendix E. "Since" gives the first Object Version that carries the resource, with the first LwM2M Enabler release that shipped that Object Version in brackets. Operations column: R/W/RW/E; "—" means the DDF `<Operations>` element is blank (no Device Management & Service Enablement access — the resource is reachable only via the Bootstrap Interface or, for Event Log 4016, only via Send).

---

## Object Model Overview

The LwM2M data model is a hierarchical tree:
```
/{ObjectID}/{InstanceID}/{ResourceID}/{ResourceInstanceID}
```

- **Object:** Defines a set of resources for a specific function (e.g., Device, Firmware Update)
- **Object Instance:** A concrete occurrence of an object. Some objects are single-instance (e.g., Device /3), others multi-instance (e.g., Server /1)
- **Resource:** A data item within an object instance (e.g., Manufacturer Name /3/0/0)
- **Resource Instance:** For multi-instance resources (e.g., Error Codes /3/0/11/*)

Resources have defined types (Core TS 1.2.2 Appendix C: String, Integer, Unsigned Integer, Float, Boolean, Opaque, Time, Objlnk, Corelnk — plus "none" for Execute-only resources, Core TS 1.2.2 Appendix D.1), operations (R/W/RW/E), instance cardinality (Single/Multiple) and optionality (Mandatory/Optional). All four identifier levels are 16-bit unsigned integers; 65535 (MAX_ID) is reserved at every level (Core TS 1.2.2 §7.4, Table 7.4-1).

---

## Core Objects (0–7)

### Object 0: LwM2M Security

Multi-instance, Mandatory. Latest Object Version **1.2** (OMA-SUP-XML_0 v1.2, `urn:oma:lwm2m:oma:0:1.2`, minimum LwM2M 1.1). One instance per server account; one instance SHOULD address the Bootstrap-Server. **No resource of this Object carries any Device Management operation** — every `<Operations>` element in the DDF is blank. Instances are provisioned only through the Bootstrap Interface (or Smartcard/factory bootstrap) and MUST NOT be accessible by any other LwM2M Server. Object 0 is never listed in the Register payload (Core TS 1.2.2 Table 6.2.1-1).

| ID | Name | Type | Instances | Mandatory | Since | Description |
|----|------|------|-----------|-----------|-------|-------------|
| 0 | LwM2M Server URI | String (0..255) | Single | Mandatory | 1.0 (1.0) | `coaps://host:port`, `coap://`, `coaps+tcp://`, `mqtts://` etc.; the scheme selects the transport binding |
| 1 | Bootstrap-Server | Boolean | Single | Mandatory | 1.0 (1.0) | True if this account is the LwM2M Bootstrap-Server |
| 2 | Security Mode | Integer (0..4) | Single | Mandatory | 1.0 (1.0) | 0 = Pre-Shared Key, 1 = Raw Public Key, 2 = Certificate, 3 = NoSec, 4 = Certificate with EST (4 added in v1.1) |
| 3 | Public Key or Identity | Opaque | Single | Mandatory | 1.0 (1.0) | Client certificate, client RPK, or the **PSK Identity** (Opaque, not String) |
| 4 | Server Public Key | Opaque | Single | Mandatory | 1.0 (1.0) | Server certificate / RPK / trust anchor, interpreted per R13 Matching Type and R15 Certificate Usage |
| 5 | Secret Key | Opaque | Single | Mandatory | 1.0 (1.0) | PSK value, or client private key |
| 6 | SMS Security Mode | Integer (0..255) | Single | Optional | 1.0 (1.0) | 0 reserved, 1 DTLS (device-terminated, PSK), 2 Secure Packet Structure (smartcard-terminated), 3 NoSec, 4 reserved, 5–203 reserved, 204–255 proprietary |
| 7 | SMS Binding Key Parameters | Opaque (6 octets) | Single | Optional | 1.0 (1.0) | KIc, KID, SPI and TAR |
| 8 | SMS Binding Secret Key(s) | Opaque (16, 32 or 48 octets) | Single | Optional | 1.0 (1.0) | Key material for the SMS binding |
| 9 | LwM2M Server SMS Number | String | Single | Optional | 1.0 (1.0) | Server MSISDN for the SMS binding |
| 10 | Short Server ID | Integer (1..65534) | Single | Optional | 1.0 (1.0) | Links to the Server Object (/1) instance; MUST be set when Bootstrap-Server = false; 0 and 65535 MUST NOT be used |
| 11 | Client Hold Off Time | Integer, s | Single | Optional | 1.0 (1.0) | Delay before Client-Initiated Bootstrap; Bootstrap-Server account only |
| 12 | Bootstrap-Server Account Timeout | Integer, s | Single | Optional | 1.0 (1.0) | Client purges the Bootstrap-Server account after this time; 0 or absent = infinite |
| 13 | Matching Type | Unsigned Integer (0..3) | Single | Optional | 1.1 (1.1) | How R4 is presented: 0 exact match (default, v1.0 behaviour), 1 SHA-256, 2 SHA-384, 3 SHA-512 hash |
| 14 | SNI | String | Single | Optional | 1.1 (1.1) | Server Name Indication for the TLS/DTLS handshake; when present R0 is the service address and SNI is used to match the certificate or PSK identity |
| 15 | Certificate Usage | Unsigned Integer (0..3) | Single | Optional | 1.1 (1.1) | Semantics of R4 (DANE-style): 0 CA constraint, 1 service certificate constraint, 2 trust anchor assertion, 3 domain-issued certificate (default when absent) |
| 16 | DTLS/TLS Ciphersuite | Unsigned Integer | Multiple | Optional | 1.1 (1.1) | Ciphersuite(s) to propose in ClientHello, IANA code points concatenated (e.g. TLS_PSK_WITH_AES_128_CCM_8 = 0xC0A8 = 49320) |
| 17 | OSCORE Security Mode | Objlnk | Single | Optional | 1.1 (1.1) | Link to an OSCORE Object (/21) instance; when present OSCORE MUST be used with that context |
| 18 | Groups To Use by Client | Unsigned Integer (0..65535) | Multiple | Optional | 1.2 (1.2) | TLS supported groups (RFC 8446 §4.2.7), ordered most- to least-preferred |
| 19 | Signature Algorithms Supported by Server | Unsigned Integer (0..65535) | Multiple | Optional | 1.2 (1.2) | RFC 8446 §4.2.3 code points |
| 20 | Signature Algorithms To Use by Client | Unsigned Integer (0..65535) | Multiple | Optional | 1.2 (1.2) | RFC 8446 §4.2.3 code points, ordered by preference |
| 21 | Signature Algorithm Certs Supported by Server | Unsigned Integer (0..65535) | Multiple | Optional | 1.2 (1.2) | Certificate-specific signature algorithms (RFC 8446 §4.2.3) |
| 22 | TLS 1.3 Features To Use by Client | Unsigned Integer (0..65535) | Single | Optional | 1.2 (1.2) | Bitmask: bit0 PSK plain, bit1 0-RTT, bit2 PSK with PFS, bit3 certificate-based authentication; bits 4–31 reserved |
| 23 | TLS Extensions Supported by Server | Unsigned Integer (0..65535) | Single | Optional | 1.2 (1.2) | Bitmap: bit0 SNI, bit1 Max Fragment Length, bit2 Status Request, bit3 Heartbeat, bit4 ALPN, bit5 SCT, bit6 Certificate Compression, bit7 Record Size Limit, bit8 Ticket Pinning, … |
| 24 | TLS Extensions To Use by Client | Unsigned Integer (0..65535) | Single | Optional | 1.2 (1.2) | Same bitmap as R23 |
| 25 | Secondary LwM2M Server URI | String (0..255) | Multiple | Optional | 1.2 (1.2) | Further URIs for the same server reachable with the same credentials (e.g. `coaps` and `coaps+tcp`) |
| 26 | MQTT Server | Objlnk | Single | Optional | 1.2 (1.2) | Link to an MQTT Server Object (/24) instance; used only when the R0 scheme is MQTT |
| 27 | LwM2M COSE Security | Objlnk | Multiple | Optional | 1.2 (1.2) | Links to LwM2M COSE Object (/23) instances |
| 28 | RDS Destination Port | Integer (0..15) | Single | Optional | 1.2 (1.2) | Default RDS destination port (3GPP TS 24.250) for the Non-IP binding via SCEF |
| 29 | RDS Source Port | Integer (0..15) | Single | Optional | 1.2 (1.2) | Default RDS source port (3GPP TS 24.250) |
| 30 | RDS Application ID | String | Single | Optional | 1.2 (1.2) | Application ID (3GPP TS 24.250) used to query the SCEF for RDS ports |

**Notes:** Object 0 v1.0 = R0–12; v1.1 added R13–17 (OMA-SUP-XML_0 v1.1); v1.2 added R18–30. Security Object instances are not accessible via the DM&SE interface. Each instance maps to one server relationship.

### Object 1: LwM2M Server

Multi-instance, Mandatory. Latest Object Version **1.2** (OMA-SUP-XML_1 v1.2, `urn:oma:lwm2m:oma:1:1.2`, minimum LwM2M 1.2). v1.0 = R0–8; v1.1 added R9–23; v1.2 added R24–27.

| ID | Name | Type | Instances | Operations | Mandatory | Since | Description |
|----|------|------|-----------|------------|-----------|-------|-------------|
| 0 | Short Server ID | Integer (1..65534) | Single | R | **Mandatory** | 1.0 (1.0) | Unique ID of this server account; matches /0/x/10 |
| 1 | Lifetime | Integer, s | Single | RW | **Mandatory** | 1.0 (1.0) | Registration lifetime; the `lt` Register parameter MUST equal this value |
| 2 | Default Minimum Period | Integer, s | Single | RW | Optional | 1.0 (1.0) | Default pmin for observations |
| 3 | Default Maximum Period | Integer, s | Single | RW | Optional | 1.0 (1.0) | Default pmax for observations |
| 4 | Disable | none | Single | E | Optional | 1.0 (1.0) | Disable this server relationship for Disable Timeout |
| 5 | Disable Timeout | Integer, s | Single | RW | Optional | 1.0 (1.0) | Period the server stays disabled |
| 6 | Notification Storing When Disabled or Offline | Boolean | Single | RW | **Mandatory** | 1.0 (1.0) | Buffer notifications while disabled/offline and deliver on reconnect |
| 7 | Binding | String | Single | RW | **Mandatory** | 1.0 (1.0) | Transport binding(s) for this server; values per Core TS 1.2.2 §6.2.1.2 — see below |
| 8 | Registration Update Trigger | none | Single | E | **Mandatory** | 1.0 (1.0) | Force a Registration Update; optional argument selects an overriding binding |
| 9 | Bootstrap-Request Trigger | none | Single | E | Optional | 1.1 (1.1) | Force a Client-Initiated Bootstrap |
| 10 | APN Link | Objlnk | Single | RW | Optional | 1.1 (1.1) | Link to the APN Connection Profile (/11) instance used for this server |
| 11 | TLS-DTLS Alert Code | Unsigned Integer (0..255) | Single | R | Optional | 1.1 (1.1) | Most recent TLS/DTLS AlertDescription received from this server (RFC 5246 §7.2) |
| 12 | Last Bootstrapped | Time | Single | R | Optional | 1.1 (1.1) | Last time the Bootstrap-Server updated this account |
| 13 | Registration Priority Order | Unsigned Integer | Single | R | Optional | 1.1 (1.1) | Registrations are sequenced in increasing order of this value |
| 14 | Initial Registration Delay Timer | Unsigned Integer, s | Single | RW | Optional | 1.1 (1.1) | Delay after the previous server in the order registers; first registration after bootstrap only |
| 15 | Registration Failure Block | Boolean | Single | R | Optional | 1.1 (1.1) | True: failure here blocks registration to later servers in the order |
| 16 | Bootstrap on Registration Failure | Boolean | Single | R | Optional | 1.1 (1.1) | True: re-bootstrap when registration is rejected or deemed failed |
| 17 | Communication Retry Count | Unsigned Integer | Single | RW | Optional | 1.1 (1.1) | Attempts per communication sequence |
| 18 | Communication Retry Timer | Unsigned Integer, s | Single | RW | Optional | 1.1 (1.1) | Base delay; multiplied by 2^(attempt-1) for exponential back-off |
| 19 | Communication Sequence Delay Timer | Unsigned Integer, s | Single | RW | Optional | 1.1 (1.1) | Delay between communication sequences; MAX_VALUE = no further sequence |
| 20 | Communication Sequence Retry Count | Unsigned Integer | Single | RW | Optional | 1.1 (1.1) | Sequences before registration is considered failed |
| 21 | Trigger | Boolean | Single | RW | Optional | 1.1 (1.1) | True: client reachable over SMS for device triggering (Execute on R8 via SMS); default false |
| 22 | Preferred Transport | String | Single | RW | Optional | 1.1 (1.1) | Single binding letter the client SHALL use to initiate the connection (e.g. switch N → U for FOTA) |
| 23 | Mute Send | Boolean | Single | RW | Optional | 1.1 (1.1) | **If true or absent, the Send operation is de-activated for this server; false activates it** |
| 24 | Alternate APN Links | Objlnk | Multiple | RW | Optional | 1.2 (1.2) | Alternate /11 instances if R10 conflicts |
| 25 | Supported Server Versions | String | Multiple | RW | Optional | 1.2 (1.2) | Enabler versions supported by the server, `1*DIGIT"."1*DIGIT["."1*DIGIT]` |
| 26 | Default Notification Mode | Integer (0..1) | Single | RW | Optional | 1.2 (1.2) | 0 = Non-Confirmable, 1 = Confirmable notifications |
| 27 | Profile ID Hash Algorithm | Unsigned Integer (0..255) | Single | RW | Optional | 1.2 (1.2) | RFC 6920 Suite ID the server prefers for dynamically generated Profile IDs |

**Binding (/1/x/7) values — Core TS 1.2.2 §6.2.1.2, Table 6.2.1.2-1.** Since v1.1 the value is a combination of transport letters only: `U` (UDP), `M` (MQTT, v1.2), `H` (HTTP, v1.2), `T` (TCP, v1.1), `S` (SMS), `N` (Non-IP, v1.1). Queue Mode is **not** part of the binding string from v1.1 onwards — it is signalled by the separate `Q` Register parameter. The v1.0 combined forms `UQ`, `SQ`, `US`, `UQS` (Core TS 1.0.2 §5.3.1.1, Table 8; `UQSQ`/`USQ` unsupported) are 1.0-only. UDP SHALL be supported by the server even if `U` is absent from R7; only one transport is used for the whole Transport Session.

### Object 2: Access Control
Multi-instance, Optional. Latest Object Version **1.1** (OMA-SUP-XML_2 v1.1). One instance per Object Instance that is accessible by more than one server.

| ID | Name | Type | Instances | Operations | Mandatory |
|----|------|------|-----------|------------|-----------|
| 0 | Object ID | Integer (1..65534) | Single | R | Mandatory |
| 1 | Object Instance ID | Integer (0..65535) | Single | R | Mandatory |
| 2 | ACL | Integer (0..31) | Multiple | RW | Optional |
| 3 | Access Control Owner | Integer (0..65535) | Single | RW | Mandatory |

ACL (/2/x/2) resource-instance ID = Short Server ID of the granted server; instance 0 holds the default rights. Bit order (OMA-SUP-XML_2 v1.1, R2): **1 = Read (also covers Observe and Write-Attributes), 2 = Write, 4 = Execute, 8 = Delete, 16 = Create**; other bits reserved. Combinations add (e.g. 3 = Read+Write). Access Control Owner = 65535 means the instance was created during Bootstrap only. Denial of an operation by ACL yields 4.01 Unauthorized.

### Object 3: Device
Single instance, Mandatory. Latest Object Version **1.2** (OMA-SUP-XML_3 v1.2). The resource set R0–22 has been stable since v1.0.

| ID | Name | Type | Instances | Operations | Mandatory |
|----|------|------|-----------|------------|-----------|
| 0 | Manufacturer | String | Single | R | Optional |
| 1 | Model Number | String | Single | R | Optional |
| 2 | Serial Number | String | Single | R | Optional |
| 3 | Firmware Version | String | Single | R | Optional |
| 4 | Reboot | none | Single | E | **Mandatory** |
| 5 | Factory Reset | none | Single | E | Optional |
| 6 | Available Power Sources | Integer (0..7) | Multiple | R | Optional |
| 7 | Power Source Voltage | Integer | Multiple | R | Optional |
| 8 | Power Source Current | Integer | Multiple | R | Optional |
| 9 | Battery Level | Integer (0..100), /100 | Single | R | Optional |
| 10 | Memory Free | Integer | Single | R | Optional |
| 11 | Error Code | Integer (0..32) | Multiple | R | **Mandatory** |
| 12 | Reset Error Code | none | Single | E | Optional |
| 13 | Current Time | Time | Single | RW | Optional |
| 14 | UTC Offset | String | Single | RW | Optional |
| 15 | Timezone | String | Single | RW | Optional |
| 16 | Supported Binding and Modes | String | Single | R | **Mandatory** |
| 17 | Device Type | String | Single | R | Optional |
| 18 | Hardware Version | String | Single | R | Optional |
| 19 | Software Version | String | Single | R | Optional |
| 20 | Battery Status | Integer (0..6) | Single | R | Optional |
| 21 | Memory Total | Integer | Single | R | Optional |
| 22 | ExtDevInfo | Objlnk | Multiple | R | Optional |

**Mandatory resources:** Reboot (/3/0/4), Error Code (/3/0/11) and Supported Binding and Modes (/3/0/16). The remaining resources are optional. Current Time (/3/0/13), UTC Offset (/3/0/14) and Timezone (/3/0/15) are RW; Reboot, Factory Reset and Reset Error Code are Execute-only.

**Enumerations (OMA-SUP-XML_3 v1.2):**
- *Available Power Sources (R6):* 0 DC, 1 Internal Battery, 2 External Battery, 3 Fuel Cell, 4 Power over Ethernet, 5 USB, 6 AC (Mains), 7 Solar. The same resource-instance ID links a power source with its voltage (R7) and current (R8).
- *Error Code (R11):* 0 No error, 1 Low battery, 2 External power supply off, 3 GPS module failure, 4 Low received signal strength, 5 Out of memory, 6 SMS failure, 7 IP connectivity failure, 8 Peripheral malfunction, 9–15 reserved, 16–32 device-specific. Instance 0 = 0 means no error; each new error creates a new instance, cleared instances are deleted.
- *Battery Status (R20), valid only when an Internal Battery (R6 = 1) exists:* 0 Normal, 1 Charging, 2 Charge Complete, 3 Damaged, 4 Low Battery, 5 Not Installed, 6 Unknown.

### Object 4: Connectivity Monitoring
Single instance, Optional. Latest Object Version **1.3** (OMA-SUP-XML_4 v1.3, package 1.3.1, 2022-12-09). v1.0 = R0–10; v1.1 added R11 SignalSNR; v1.2 added R12 LAC; v1.3 added R13 Coverage Enhancement Level.

| ID | Name | Type | Instances | Operations | Mandatory |
|----|------|------|-----------|------------|-----------|
| 0 | Network Bearer | Integer (0..50) | Single | R | Mandatory |
| 1 | Available Network Bearer | Integer (0..50) | Multiple | R | Mandatory |
| 2 | Radio Signal Strength | Integer, dBm | Single | R | Mandatory |
| 3 | Link Quality | Integer | Single | R | Optional |
| 4 | IP Addresses | String | Multiple | R | Mandatory |
| 5 | Router IP Addresses | String | Multiple | R | Optional |
| 6 | Link Utilization | Integer (0..100), % | Single | R | Optional |
| 7 | APN | String | Multiple | R | Optional |
| 8 | Cell ID | Integer | Single | R | Optional |
| 9 | SMNC | Integer (0..999) | Single | R | Optional |
| 10 | SMCC | Integer (0..999) | Single | R | Optional |
| 11 | SignalSNR | Integer, dB | Single | R | Optional |
| 12 | LAC | Integer | Single | R | Optional |
| 13 | Coverage Enhancement Level | Integer (0..4) | Single | R | Optional |

The DDF name of R6 is "Link Utilization" (US spelling, as registered); R11 SignalSNR is the SINR. R11 is declared **Integer** in OMA-SUP-XML_4 v1.3.

**Network Bearer enum (R0/R1):** 0–20 cellular, 21–40 local wireless, 41–50 wireline. 0 GSM, 1 TD-SCDMA, 2 WCDMA, 3 CDMA2000, 4 WiMAX, 5 LTE-TDD, 6 LTE-FDD, 7 NB-IoT, 8 5G-NR TDD, 9 5G-NR FDD, 10–20 reserved cellular; 21 WLAN, 22 Bluetooth, 23 IEEE 802.15.4, 24–40 reserved wireless; 41 Ethernet, 42 DSL, 43 PLC, 44–50 reserved wireline.

**Coverage Enhancement Level (R13, 3GPP TS 36.331/36.213):** 0 no CE in serving cell, 1 CE level 0, 2 CE level 1, 3 CE level 2, 4 CE level 3.

### Object 5: Firmware Update
**Single instance**, Optional (OMA-SUP-XML_5; Core TS 1.2.2 Table E.6-1). Latest Object Version **1.2** (Core TS 1.2.2 Appendix E.6, `urn:oma:lwm2m:oma:5:1.2`, minimum LwM2M 1.1); OMA-SUP-XML_5 v1.1 (2020-11-10) carries R0–13, v1.2 adds R14. v1.0 = R0–9. Because the Object is single-instance, a device with several independently updatable components uses Software Management (/9) and Software Component (/14), not multiple /5 instances.

| ID | Name | Type | Instances | Operations | Mandatory | Since |
|----|------|------|-----------|------------|-----------|-------|
| 0 | Package | Opaque | Single | W | Mandatory | 1.0 |
| 1 | Package URI | String (0..255) | Single | **RW** | Mandatory | 1.0 |
| 2 | Update | none | Single | E | Mandatory | 1.0 |
| 3 | State | Integer (0..3) | Single | R | Mandatory | 1.0 |
| 5 | Update Result | Integer (0..11) | Single | R | Mandatory | 1.0 |
| 6 | PkgName | String (0..255) | Single | R | Optional | 1.0 |
| 7 | PkgVersion | String (0..255) | Single | R | Optional | 1.0 |
| 8 | Firmware Update Protocol Support | Integer (0..5) | Multiple | R | Optional | 1.0 |
| 9 | Firmware Update Delivery Method | Integer (0..2) | Single | R | Mandatory | 1.0 |
| 10 | Cancel | none | Single | E | Optional | 1.1 |
| 11 | Severity | Integer (0..2) | Single | RW | Optional | 1.1 |
| 12 | Last State Change Time | Time | Single | R | Optional | 1.1 |
| 13 | Maximum Defer Period | Unsigned Integer, s | Single | RW | Optional | 1.1 |
| 14 | Automatic Upgrade at Download | Boolean | Single | RW | Optional | 1.2 |

(Resource ID 4 is unassigned.)

**State machine (Core TS 1.2.2 E.6, R3 description):** 0 Idle → 1 Downloading → 2 Downloaded → 3 Updating → 0 Idle on success. Update (R2) is executable **only in Downloaded**. Writing an empty string to Package URI or NULL (`\0`) to Package resets the machine to Idle with Update Result 0. If Update fails the state returns to Downloaded **or** Idle depending on the cause (integrity failure → Idle); a failure never leaves the client in Updating. Cancel (R10) is allowed only before the update has started (4.05 Method Not Allowed otherwise) and sets Update Result 10, State 0. A deferred update stays in Downloaded with Update Result 11 until the user accepts or Maximum Defer Period (R13) expires. Automatic Upgrade at Download (R14) = true means the device proceeds to Updating without a server Execute.

**Delivery method (R9):** 0 = Pull only, 1 = Push only, 2 = Both.

**Protocol Support (R8):** 0 CoAP (+block-wise; default), 1 CoAPS, 2 HTTP 1.1, 3 HTTPS 1.1, 4 CoAP over TCP (RFC 8323), 5 CoAP over TLS (RFC 8323). A server MUST NOT write a Package URI using a scheme the client did not advertise. **Severity (R11):** 0 Critical, 1 Mandatory (default), 2 Optional.

#### Firmware Update — State and Update Result

**State (/5/0/3):**

| Value | State |
|-------|-------|
| 0 | Idle |
| 1 | Downloading |
| 2 | Downloaded |
| 3 | Updating |

**Update Result (/5/0/5):**

| Value | Result |
|-------|--------|
| 0 | Initial (no firmware update yet, or after a successful reset); MUST be reset to 0 when a download/update starts |
| 1 | Firmware updated successfully |
| 2 | Not enough flash |
| 3 | Out of RAM |
| 4 | Connection lost during download |
| 5 | Integrity check failure |
| 6 | Unsupported package type |
| 7 | Invalid URI |
| 8 | Firmware update failed |
| 9 | Unsupported protocol (URI scheme not advertised in R8) |
| 10 | Firmware update cancelled (Cancel executed successfully) |
| 11 | Firmware update deferred |

**Delivery Method (/5/0/9):** 0=Pull, 1=Push, 2=Both.
- **PULL** — the server writes the Package URI to /5/0/1 and the client fetches the binary itself.
- **PUSH** — the server writes the binary directly to /5/0/0 Package (Opaque), using CoAP block-wise transfer (mandatory for clients implementing Object 5).

A conformant client sets **Update Result** (not just State) on every terminal outcome, so the server can distinguish success from each failure cause.

### Object 6: Location
Single instance, Optional. Object Version **1.0** (OMA-SUP-XML_6 v1.0). GPS/GNSS data.

| ID | Name | Type | Instances | Operations | Mandatory |
|----|------|------|-----------|------------|-----------|
| 0 | Latitude | Float, lat | Single | R | Mandatory |
| 1 | Longitude | Float, lon | Single | R | Mandatory |
| 2 | Altitude | Float, m | Single | R | Optional |
| 3 | Radius | Float, m | Single | R | Optional |
| 4 | Velocity | Opaque (3GPP TS 23.032 encoding) | Single | R | Optional |
| 5 | Timestamp | Time | Single | R | Mandatory |
| 6 | Speed | Float, m/s | Single | R | Optional |

### Object 7: Connectivity Statistics
Single instance, Optional. Object Version **1.0** (OMA-SUP-XML_7 v1.0). Data usage tracking.

| ID | Name | Type | Instances | Operations | Mandatory |
|----|------|------|-----------|------------|-----------|
| 0 | SMS Tx Counter | Integer | Single | R | Optional |
| 1 | SMS Rx Counter | Integer | Single | R | Optional |
| 2 | Tx Data | Integer | Single | R | Optional |
| 3 | Rx Data | Integer | Single | R | Optional |
| 4 | Max Message Size | Integer, B | Single | R | Optional |
| 5 | Average Message Size | Integer, B | Single | R | Optional |
| 6 | Start | none | Single | E | Mandatory |
| 7 | Stop | none | Single | E | Mandatory |
| 8 | Collection Period | Integer, s | Single | RW | Optional |

Start (R6, Execute) resets R0–5 to 0 and begins collection; if Collection Period (R8) is 0 collection runs until Stop (R7, Execute), otherwise it ends after the period. Stop does not reset the counters. Tx/Rx Data report total kilobytes over IP bearers including headers up to the IP layer (per the R6 description; the DDF Units element is blank), excluding RAN retransmissions and SMS.

---

## Extended OMA object resource maps

### Object 8: Lock and Wipe

Single instance, Optional. Object Version **1.0** (OMA-SUP-XML_8, LockWipe enabler 1.0.4, 2024-03-12).

| ID | Name | Type | Instances | Operations | Mandatory |
|----|------|------|-----------|------------|-----------|
| 0 | State | Integer (0..2) | Single | RW | Mandatory |
| 1 | Lock target | String | Multiple | W | Mandatory |
| 2 | Wipe item | String | Multiple | R | Optional |
| 3 | Wipe | none | Single | E | Mandatory |
| 4 | Wipe target | String | Multiple | W | Mandatory |
| 5 | Lock or Wipe Operation Result | Integer (0..8) | Single | R | Mandatory |

**State (R0):** 0 unlocked, 1 partially locked (targets in R1), 2 fully locked. **Operation Result (R5):** 0 Default, 1 Partial lock OK, 2 Full lock OK, 3 Unlock OK, 4 Wipe OK, 5 Partial lock failed, 6 Full lock failed, 7 Unlock failed, 8 Wipe failed.

### Object 9: Software Management

Multi-instance — one instance per software package. Object Version **1.0** (OMA-SUP-XML_9, SwMgmt enabler 1.0.3, 2024-03-12). Distinct from /5, which owns the single firmware/OS image; /9 manages installable application software packages, optionally decomposed into Software Component (/14) instances.

| ID | Name | Type | Instances | Operations | Mandatory | Description |
|----|------|------|-----------|------------|-----------|-------------|
| 0 | PkgName | String (0..255) | Single | R | Mandatory | Name of the software package |
| 1 | PkgVersion | String (0..255) | Single | R | Mandatory | Version of the software package |
| 2 | Package | Opaque | Single | W | Optional | Software binary delivered by push |
| 3 | Package URI | String (0..255) | Single | **W** | Optional | URI the client fetches the package from (pull) |
| 4 | Install | none | Single | E | Mandatory | Install the delivered package |
| 5 | Checkpoint | Objlnk | Single | R | Optional | Link to a checkpoint object |
| 6 | Uninstall | none | Single | E | Mandatory | Uninstall the package (argument 0 = "for update") |
| 7 | Update State | Integer (0..4) | Single | R | Mandatory | Software update state machine |
| 8 | Update Supported Objects | Boolean | Single | RW | Optional | Whether supported objects are updated on install |
| 9 | Update Result | Integer (0..200) | Single | R | Mandatory | Result code of the last download/install/uninstall |
| 10 | Activate | none | Single | E | Mandatory | Activate the installed software |
| 11 | Deactivate | none | Single | E | Mandatory | Deactivate the installed software |
| 12 | Activation State | Boolean | Single | R | Mandatory | 0 DISABLED / 1 ENABLED |
| 13 | Package Settings | Objlnk | Single | RW | Optional | Link to package-specific settings object |
| 14 | User Name | String (0..255) | Single | W | Optional | Credential for pull-mode download |
| 15 | Password | String (0..255) | Single | W | Optional | Credential for pull-mode download |
| 16 | Status Reason | String | Single | R | Optional | Status of component actions (SwMgmt TS Appendix B) |
| 17 | Software Component Link | Objlnk | Multiple | R | Optional | Links to /14 instances; when present R2, R3 and R12 are ignored |
| 18 | Software Component tree length | Integer (0..255) | Single | R | Optional | Number of /14 instances for this package |

**Update State (R7):** 0 INITIAL, 1 DOWNLOAD STARTED, 2 DOWNLOADED, 3 DELIVERED (ready to install; a failed Install stays here), 4 INSTALLED (Activate/Deactivate apply). Uninstall returns to INITIAL. **Update Result (R9):** 0 Initial, 1 Downloading, 2 Installed successfully, 3 Downloaded and integrity verified, 4–49 reserved, 50 Not enough storage, 51 Out of memory, 52 Connection lost, 53 Integrity check failure, 54 Unsupported package type, 56 Invalid URI, 57 Device-defined update error, 58 Installation failure, 59 Uninstallation failure during forUpdate(arg=0), 60–200 reserved. **Activation State (R12):** 0 DISABLED, 1 ENABLED. A SOTA flow therefore observes /9/x/7 (Update State) and /9/x/9, executes /9/x/4 Install then /9/x/10 Activate, and confirms via /9/x/12.

### Object 11: APN Connection Profile

Multi-instance — one instance per APN profile. Object Version **1.1** (OMA-SUP-XML_11 v1.1, minimum LwM2M 1.1; v1.0 = R0–25, v1.1 added R26–33). Linked from the Server Object via /1/x/10 APN Link and /1/x/24 Alternate APN Links.

| ID | Name | Type | Instances | Operations | Mandatory | Description |
|----|------|------|-----------|------------|-----------|-------------|
| 0 | Profile name | String | Single | RW | Mandatory | Human-readable profile name |
| 1 | APN | String | Single | RW | Optional | Access Point Name |
| 2 | Auto select APN by device | Boolean | Single | RW | Optional | Device selects APN automatically |
| 3 | Enable status | Boolean | Single | RW | Optional | Whether this profile is enabled |
| 4 | Authentication Type | Integer | Single | RW | Mandatory | 0 PAP, 1 CHAP, 2 PAP or CHAP, 3 None |
| 5 | User Name | String | Single | RW | Optional | Authentication user name |
| 6 | Secret | String | Single | **RW** | Optional | Authentication secret (e.g. PAP); the DDF declares RW — restrict read-back by ACL/policy if required |
| 7 | Reconnect Schedule | String | Single | RW | Optional | Schedule for reconnection attempts |
| 8 | Validity (MCC, MNC) | String | Multiple | **RW** | Optional | Comma-separated MCC,MNC for which the profile is valid |
| 9 | Connection establishment time | Time | Multiple | R | Optional | Per-connection timestamp |
| 10 | Connection establishment result | Integer | Multiple | R | Optional | 0 accepted, 1 rejected |
| 11 | Connection establishment reject cause | Integer (0..111) | Multiple | R | Optional | 3GPP TS 24.008 / 24.301 cause |
| 12 | Connection end time | Time | Multiple | R | Optional | Per-connection teardown time |
| 13 | TotalBytesSent | Integer | Single | R | Optional | Cumulative bytes transmitted |
| 14 | TotalBytesReceived | Integer | Single | R | Optional | Cumulative bytes received |
| 15 | IP address | String | Multiple | RW | Optional | IPv4 or IPv6 address(es) |
| 16 | Prefix length | String | Multiple | RW | Optional | |
| 17 | Subnet mask | String | Multiple | RW | Optional | |
| 18 | Gateway | String | Multiple | RW | Optional | |
| 19 | Primary DNS address | String | Multiple | RW | Optional | |
| 20 | Secondary DNS address | String | Multiple | RW | Optional | |
| 21 | QCI | Integer (1..9) | Single | R | Optional | QoS Class Identifier |
| 22 | Vendor specific extensions | Objlnk | Single | R | Optional | |
| 23 | TotalPacketsSent | Integer | Single | R | Optional | |
| 24 | PDN Type | Integer | Single | RW | Optional | 0 Non-IP, 1 IPv4, 2 IPv6, 3 IPv4v6 — preferred binding when listed in /1/x/7 (Core TS 1.2.2 §6.2.1.2) |
| 25 | APN Rate Control | Integer | Single | R | Optional | |
| 26 | Serving PLMN Rate control | Unsigned Integer | Single | RW | Optional | Max NAS messages per Uplink Time Unit per PDN (3GPP TS 23.401 §4.7.7.2); moved here from /10 |
| 27 | Uplink Time Unit | Unsigned Integer (0..4) | Single | RW | Optional | 0 unrestricted, 1 minute, 2 hour, 3 day, 4 week |
| 28 | APN Rate Control for Exception Data | Integer | Single | R | Optional | |
| 29 | APN Exception Data Uplink Time Unit | Unsigned Integer (0..4) | Single | RW | Optional | |
| 30 | Supported RAT Types | Unsigned Integer | Multiple | RW | Optional | 1 3GPP PS, 2 GSM/GPRS, 3 UMTS, 4 LTE, 5 1xEV-DO, 6 3GPP CS, 12 LTE CIoT UP, 13 LTE CIoT CP, 14 NB-IoT CP, 15 NB-IoT UP, 16–100 reserved |
| 31 | RDS Application ID | String | Multiple | RW | Optional | 3GPP TS 24.250 |
| 32 | RDS Destination Port | Integer (0..15) | Multiple | RW | Optional | |
| 33 | RDS Source Port | Integer (0..15) | Multiple | RW | Optional | |

**Note:** OMA-SUP-XML_11 v1.1 declares Secret (/11/x/6) as **RW**, not Write-only; a deployment that must prevent credential read-back enforces that through ACL or server policy, not through the DDF.

### Object 25: LwM2M Gateway

Multi-instance — one instance per IoT Device connected behind the gateway. Defined in the OMA LwM2M Gateway Technical Specification (OMA-TS-LWM2M_Gateway; v1.1.1 Approved 2024-03-12). This is the canonical pattern for exposing non-LwM2M or constrained end devices through a single registered LwM2M Client: the gateway is the only registered endpoint, the server addresses each end device's Objects under the instance's Prefix, and the gateway acts on the device's behalf with or without protocol translation. The Gateway TS 1.1.1 §4 note is explicit that the gateway **does not forward LwM2M messages natively to the IoT Device and is therefore different from an IP router or a CoAP proxy**.

**Object 25 v2.0 (OMA-SUP-XML_25 v2.0, 2024-03-12, Gateway TS 1.1/1.1.1; `urn:oma:lwm2m:oma:25:2.0`, minimum LwM2M 1.1):**

| ID | Name | Type | Instances | Operations | Mandatory | Description |
|----|------|------|-----------|------------|-----------|-------------|
| 0 | Device ID | String | Single | R | Mandatory | Identifies the IoT Device connected to the gateway |
| 1 | Prefix | String | Single | **R** | Mandatory | Prefix that MUST be used to access this device's Objects |
| 3 | IoT Device Objects | **Corelnk** | Single | R | Mandatory | Objects/Object Instances exposed on behalf of the device, same CoRE Link format as the Register payload (e.g. `</3/0>,</3303/0>`) |

Resource ID 2 is not used in v2.0. **Object 25 v1.0** (Core TS 1.2.2 Appendix E.12; OMA-SUP-XML_25 v1.0, 2020-11-10, minimum LwM2M 1.2) had: 0 Device ID (String, R, Mandatory), 1 Prefix (String, **RW**, Mandatory), 2 Routing Table Entry (Objlnk, R, Optional) pointing at a Gateway Routing (/26) instance. v2.0 is a Type I (major) change: Prefix operations and the resource set changed, and Object 26 is no longer referenced by the Gateway TS 1.1.1.

### Object 20: Event Log — version history

Object 20 (Event Log, `urn:oma:lwm2m:oma:20[:ver]`) uses Reusable Resource IDs 4010–4017. Latest Object Version **3.0** (OMA-SUP-XML_LWM2M_EventLog v3.0, 2025-07-14, minimum LwM2M 1.1).

| ObjVer | Date | Instances | Resources | Change |
|--------|------|-----------|-----------|--------|
| 1.0 (1.0.1) | 2020-06-16 | Single | 4010–4015 | Initial: LogClass (RW), LogStart (E), LogStop (E), LogStatus (R, bitfield), LogData (R, Opaque, Mandatory), LogDataFormat (RW: 0 raw, 1 TLV, 2 JSON, 3 CBOR, 100–255 vendor) |
| 2.0 | 2023-01-31 | **Multiple** | 4010–4015 | Instances changed Single → Multiple (Type I) |
| 2.1 | 2024-01-23 | Multiple | + 4016, 4017 | 4016 LogDataIncremental (Opaque, **blank Operations — Send-only**; DM access returns 4.05), 4017 LogDataIncrementalMinimumSendInterval (Unsigned Integer, RW; 0/absent = never sent) |
| 3.0 | 2025-07-14 | Multiple | 4010–4017 | LogDataFormat adds 4 Syslog (RFC 5424) and 5 Common Log Format (RFC 6872); 6–99 reserved |

**LogClass (4010):** 0 generic, 1 system, 2 security, 3 event, 4 trace, 5 panic, 6 charging, 7–99 reserved, 100–255 vendor. **LogStatus (4013):** bit0 0=running/1=stopped, bit1 LogData valid, bit2 error during collection, bits 3–6 reserved, bit7 vendor. **Spec defect:** the EventLog 3.0 TS text (OMA-TS-LWM2M_EventLog-V3_0-20250714-A) contains an unresolved merge-conflict block (`<<<<<<<` … `>>>>>>>`) inside the 4015 LogDataFormat row; both branches agree on values 3–5. Treat the OMA-SUP-XML_LWM2M_EventLog v3.0 file as authoritative.

### Object 21: OSCORE

Multi-instance, Optional. Latest Object Version **2.0** (OMA-SUP-XML_21 v2.0, 2021-11-23, minimum LwM2M 1.1). Introduced with LwM2M 1.1; linked from /0/x/17. All `<Operations>` are blank — Bootstrap-only, never exposed over DM&SE and never in the Register payload. v1.0 = R0–5 (all key material typed **String**); v1.1 added R6 ID Context (String); **v2.0 retyped R0, R1, R2, R5 and R6 to Opaque** (Type I change) so that binary identifiers and secrets are carried without text encoding.

| ID | Name | Type (v2.0) | Instances | Mandatory | Description |
|----|------|-------------|-----------|-----------|-------------|
| 0 | OSCORE Master Secret | Opaque | Single | Mandatory | Master Secret of the Common Context |
| 1 | OSCORE Sender ID | Opaque | Single | Mandatory | |
| 2 | OSCORE Recipient ID | Opaque | Single | Mandatory | |
| 3 | OSCORE AEAD Algorithm | Integer | Single | Optional | COSE algorithm ID (RFC 8152 Table 10) |
| 4 | OSCORE HMAC Algorithm | Integer | Single | Optional | HKDF HMAC algorithm (RFC 8152 Table 7) |
| 5 | OSCORE Master Salt | Opaque | Single | Optional | |
| 6 | OSCORE ID Context | Opaque | Single | Optional | ID Context used to identify the Common Context (added v1.1) |

### Object 15: DevCapMgmt — v2.0 change

Multi-instance, Optional. Latest Object Version **2.0** (OMA-SUP-XML_15 v2.0, DevCapMgmt enabler 2.0.1, 2024-03-12). Resources: 0 Property (String, R, Mandatory), 1 Group (Integer 0..15, R, Mandatory: 0 SENSOR, 1 CONTROL, 2 CONNECTIVITY, 3 NAVIGATION, 4 STORAGE, 5 VISION, 6 SOUND, 7 ANALOG_INPUT, 8 ANALOG_OUTPUT, 9–15 reserved), 2 Description (String, R), 3 Attached (Boolean, R), 4 Enabled (Boolean, R, Mandatory), 5 opEnable (E, Mandatory), 6 opDisable (E, Mandatory), 7 NotifyEn (Boolean, RW). The v1.0 DDF declared opDisable (R6) with `MultipleInstances = Multiple`, which violates the rule that an Execute resource MUST be Single (Core TS 1.2.2 Appendix D.1); v2.0 corrects R6 to Single — a change of a resource's "Instances" characteristic, hence Type I and a major version bump. The resource set is otherwise unchanged.

---

## Extended OMA Objects (8–28)

Objects 8–20 were produced by separate OMA companion enablers, not by the LwM2M Core enabler, and are versioned independently; Objects 21, 23–27 are carried in Core TS 1.2.2 Appendix E (E.9–E.14) at the version current when 1.2 was released; Object 22 comes from the Virtual Observe Notify enabler; Object 28 exists only in the OMNA registry. "Min LwM2M (DDF)" is the `<LWM2MVersion>` element of the latest DDF (omitted = 1.0 per Core TS 1.2.2 Appendix J).

| ID | Name | Latest ObjVer | Min LwM2M (DDF) | Defining spec / date | Key purpose |
|----|------|---------------|-----------------|----------------------|-------------|
| 8 | Lock and Wipe | 1.0 | 1.0 | OMA LwM2M LockWipe 1.0.4, 2024-03-12 | Remote device lock/wipe for theft protection |
| 9 | LwM2M Software Management | 1.0 | 1.0 | OMA LwM2M SwMgmt 1.0.3, 2024-03-12 | Download/install/activate software packages |
| 10 | LwM2M Cellular Connectivity | 1.1 | 1.1 | OMA LwM2M ConnMgmt 1.0 (2017-03-14, status Historic); object continued as standalone release OMA-SUP-XML_10 v1.1, 2020-11-10 | SMSC address, disable-radio period, PSM/Active timers, eDRX parameters per RAT mode, Activated Profile Names (links to /11), power-saving modes, RAI usage. APN configuration lives in /11; RAT preference in /13 |
| 11 | LwM2M APN Connection Profile | 1.1 | 1.1 | ConnMgmt 1.0 (Historic); standalone OMA-SUP-XML_11 v1.1, 2020-11-10 | Per-APN configuration: authentication, PDN type, addressing, rate control, RAT types, connection statistics |
| 12 | WLAN connectivity | 1.1 (pkg 1.1.1) | 1.0 | ConnMgmt 1.0 (Historic); standalone OMA-SUP-XML_12 v1.1.1, 2022-10-03 | Wi-Fi interface configuration and statistics (R0–48) |
| 13 | LwM2M Bearer Selection | 1.1 | 1.1 | ConnMgmt 1.0 (Historic); standalone OMA-SUP-XML_13 v1.1, 2020-11-10 | Preferred bearer list (0 auto … 16 5G-NR), acceptable RSSI/RSCP/RSRP thresholds, PLMN lists |
| 14 | LwM2M Software Component | 1.0 | 1.0 | OMA LwM2M SwMgmt 1.0.3, 2024-03-12 | Per-component identity, pack, version, Activate/Deactivate, Activation State; linked from /9/x/17 |
| 15 | DevCapMgmt | 2.0 | 1.0 | OMA LwM2M DevCapMgmt 2.0.1, 2024-03-12 | Enable/disable device capabilities (sensors, radios, storage …) and attach/detach notification |
| 16 | Portfolio | 1.0 | 1.0 | OMA LwM2M PortfolioObj 1.0.1, 2020-06-16 | Data-storage extension for other Object Instances plus authentication services: Identity (RW, multi; e.g. GSMA host device ID/manufacturer/model/software version), GetAuthData (E), AuthData (R, Opaque, multi), AuthStatus (R: 0 IDLE, 1 DATA_AVAIL, 2 ERROR) |
| 17 | Communications Characteristics | 1.0 | — (no DDF published) | OMNA registry entry only (Owner OMA) | Generic communications parameter configuration |
| 18 | Non-Access Stratum (NAS) Configuration | 1.0 | — (no DDF published) | OMNA registry entry only (Owner OMA); derived from 3GPP TS 24.368 | NAS configuration parameters |
| 19 | BinaryAppDataContainer | 1.0 | 1.0 | OMA LwM2M BinaryAppDataCont 1.0.2, 2024-03-12 | Opaque application data exchange (Data RW multi, priority, creation time, description, format, App ID) |
| 20 | Event Log | 3.0 | 1.1 | OMA LwM2M EventLog 3.0, 2025-07-14 | Device log collection (see version history above) |
| 21 | LwM2M OSCORE | 2.0 | 1.1 | Core TS 1.2.2 Appendix E.9; OMA-SUP-XML_21 v2.0, 2021-11-23 | OSCORE security context (introduced with LwM2M 1.1) |
| 22 | Virtual Observe Notify | 1.1 | 1.1 | OMA LwM2M VirtualObserveNotify 1.0 (Approved 2019-07-11); v1.1 Guidelines draft 2021-06-21; DDF 22.xml v1.1 | Observe/notify multiple resources across objects/instances in one notification (v1.1 adds Report Format and Opaque Report) |
| 23 | LwM2M COSE | 1.0 | 1.2 | Core TS 1.2.2 Appendix E.10; OMA-SUP-XML_23 v1.0, 2020-11-10 | COSE key material (KID, AEAD Algorithm, Key) for payload protection, e.g. with the MQTT binding; Bootstrap-only, excluded from Register |
| 24 | MQTT Server | 1.0 | 1.2 | Core TS 1.2.2 Appendix E.11; OMA-SUP-XML_24 v1.0, 2020-11-10 | MQTT broker session parameters (Will, Clean Session, Keep Alive, Client Identifier, credentials); linked from /0/x/26 |
| 25 | LwM2M Gateway | 2.0 | 1.1 | v1.0: Core TS 1.2.2 Appendix E.12 (2020-11-10). v2.0: OMA LwM2M Gateway TS 1.1/1.1.1, 2024-03-12 | One instance per IoT Device behind the gateway: Device ID, Prefix, IoT Device Objects (Corelnk). See the resource map under "Object 25: LwM2M Gateway" above |
| 26 | LwM2M Gateway Routing | 1.0 | 1.2 | Core TS 1.2.2 Appendix E.13; OMA-SUP-XML_26 v1.0, 2020-11-10 | Routing table for Object 25 v1.0: LwM2M Object ID (Unsigned Integer) → Mapping Info (Corelnk; empty = identity). Not referenced by Gateway TS 1.1.1 / Object 25 v2.0 |
| 27 | 5GNR Connectivity | 1.0 | 1.2 | Core TS 1.2.2 Appendix E.14; OMA-SUP-XML_27 v1.0, 2020-11-10 | 5G NR connectivity: Connectivity Option (1/3 EPC+LTE[+NR], 2/4 5GC+NR[+eLTE], 5/7 5GC+eLTE[+NR]), NR bands, S-NSSAI, DNN, PDU session parameters, QoS |
| 28 | Device RF Capabilities | 1.0 | 1.1 | OMNA registry DDF 28.xml (Owner OMA) | Supported band information per RAT (LTE, EGPRS, TD-SCDMA, WCDMA, CDMA2000, WiMAX, NB-IoT, 5G) |

> **Source of truth:** the IDs/names above are reconciled against the OMNA LwM2M Object registry (`openmobilealliance.org/specifications/registries/objects`, mirrored at `OpenMobileAlliance/lwm2m-registry` → `prod/DDF.xml`) and the object DDF files listed. Object **23 = LwM2M COSE**, **24 = MQTT Server**, **25 = LwM2M Gateway**, **26 = LwM2M Gateway Routing** — confirmed against Core TS 1.2.2 Appendix E.10–E.13. (An earlier revision had the 22–26 block scrambled; there is **no** "reserved/unassigned" gap at 23.) Objects 17 and 18 are registered names without a published DDF in the registry mirror — verify against OMNA before relying on their resource sets. The ConnMgmt enabler (Objects 10–13) is status Historic; its objects continue as standalone object releases.

---

## IPSO Smart Objects

After the IPSO Alliance merged with OMA in 2018, IPSO Smart Objects became part of the OMNA LwM2M Registry. These are generic sensor/actuator objects registered in the SDO (`ext`) class — registry owner "IPSO Alliance" for 3200–3350 (`urn:oma:lwm2m:ext:<id>`); 3351–3386 are GSMA and 3387–3406 OpenAIS allocations:

| Object ID | Name | Typical Use |
|-----------|------|-------------|
| 3300 | Generic Sensor | Custom sensor value |
| 3301 | Illuminance Sensor | Lux measurement |
| 3302 | Presence Sensor | Occupancy detection |
| 3303 | Temperature Sensor | Temperature in °C |
| 3304 | Humidity Sensor | Relative humidity % |
| 3305 | Power Measurement | Watts, cumulative energy |
| 3306 | Actuation | On/Off actuator |
| 3308 | Set Point | Target temperature, etc. |
| 3310 | Load Control | Demand response |
| 3311 | Light Control | Dimmer, colour |
| 3312 | Power Control | Power relay |
| 3313 | Accelerometer | 3-axis acceleration |
| 3314 | Magnetometer | 3-axis magnetic field |
| 3315 | Barometer | Atmospheric pressure |
| 3316 | Voltage | Voltage measurement |
| 3317 | Current | Current measurement |
| 3318 | Frequency | Frequency measurement |
| 3319 | Depth | Distance/depth measurement |
| 3320 | Percentage | Generic percentage |
| 3321 | Altitude | Height above reference |
| 3322 | Load | Weight/force measurement |
| 3323 | Pressure | Generic pressure |
| 3324 | Loudness | Sound level dB |
| 3325 | Concentration | Gas/particle concentration |
| 3326 | Acidity | pH measurement |
| 3327 | Conductivity | Electrical conductivity |
| 3328 | Power | Power generation/consumption |
| 3329 | Power Factor | Power factor |
| 3330 | Distance | Distance measurement |
| 3331 | Energy | Energy measurement |
| 3332 | Direction | Compass heading |
| 3333 | Time | Time reference |
| 3334 | Gyrometer | 3-axis angular velocity |
| 3335 | Colour | RGB/RGBW colour |
| 3336 | GPS Location | GPS coordinates |
| 3337 | Positioner | Servo/motor position |
| 3338 | Buzzer | Audible alert |
| 3339 | Audio Clip | Audio file reference |
| 3340 | Timer | Countdown/interval timer |
| 3341 | Addressable Text Display | LCD/LED text display |
| 3342 | On/Off Switch | Binary input |
| 3343 | Level Control | Slider/level input |
| 3344 | Up/Down Control | Increment/decrement |
| 3345 | Multiple Axis Joystick | Multi-axis input |
| 3346 | Rate | Rate of change |
| 3347 | Push Button | Digital input |
| 3348 | Multi-state Selector | Multiple-choice input |
| 3349 | Bitmap | Bit-field input/output |
| 3350 | Stopwatch | Elapsed time |

---

## Industry-Registered Objects

Selected notable third-party registrations. The normative class boundaries are in Core TS 1.2.2 Table D.2.1-1 (next section); the sub-bands below are **registry observations**, not spec-defined carve-outs — an owner's objects are simply the IDs OMNA allocated to it in sequence. Owners and IDs below were checked against a mirror of `prod/DDF.xml`; confirm against the live OMNA registry before quoting.

| Object IDs | Class / URN segment | Organisation | Domain |
|------------|---------------------|--------------|--------|
| 500–510 | OMA (`oma`) | OMA | CoAP Config (500), LTE-MTC Band Config (501), CO Detector (502), Fire Alarm (503), **Remote SIM Provisioning (504)**, LwM2M SIM Provisioning (505), nuSIM Profile/nuSIM (507/508), Measurement Metadata (509), Measurement Quality Reason (510). All ObjVer 1.0; registered objects, not v2.0 deliverables |
| 2048–2057 | SDO (`ext`) | oneM2M | CMDH policy objects (CmdhPolicy, ActiveCmdhPolicy, CmdhDefaults, … CmdhBackOffParametersSet) for oneM2M interworking |
| 3200–3350 | SDO (`ext`) | IPSO Alliance (now OMA) | IPSO Smart Objects (digital/analogue I/O 3200–3203, sensors and actuators 3300–3350) |
| 3351–3386 | SDO (`ext`) | GSMA | GSMA IoT objects |
| 3387–3406 | SDO (`ext`) | OpenAIS | Lighting (oA Physical Presence Sensor, Push-Button, Scene, Status Report …) |
| 3407–3408 | SDO (`ext`) | IoT Connectivity Alliance | Smoke Alarm, Manhole Cover |
| 3410–3439, 3445–3449 | SDO (`ext`) | uCIFI Alliance | Smart City: Device Extension, Battery, LPWAN Communication, Outdoor lamp controller, Electrical monitor, meters (electricity/gas/water), Air quality, Parking, Traffic Counter, Noise, People counter, Filling level, Edge computing (3436–3439), Stream/Rain Gauge, LPWAN Mesh, Anemometer — 66 registry rows across versions |
| 3441–3442 | SDO (`ext`) | Eclipse Foundation | LwM2M v1.0 / v1.1 Test Objects (Leshan) |
| **3443** | SDO (`ext`) | **GSMA** | **eSIM IoT** (`urn:oma:lwm2m:ext:3443`, ObjVer 1.0): eIM↔IPA interface per GSMA SGP.32 — State, EID, eIM message (W, Opaque), IPA message (R, Opaque), Clear IPA message (E). Not a uCIFI object and not a v2.0 deliverable |
| 10262–10284 | Vendor (`x`) | South East Water Corporation | "SEW Digital Utility IoT LwM2M Technical Specification" — water metering |
| 10300–10399 | Vendor (`x`) | Various (AT&T, AVSystem, Ericsson, Itron, Sierra Wireless, Vaisala, …) | Vendor device-management and application objects; no single domain |
| 18830–18831 | Vendor (`x`) | IoTerop | MQTT Broker, MQTT Publication |

The uCIFI band is **not** "carved out of the IPSO range": IPSO-owned IDs end at 3350 and the 3351–3449 block holds ordinary sequential SDO allocations to several owners (OpenAIS, IoT Connectivity Alliance, uCIFI, Eclipse, GSMA). Object 3443 in particular is GSMA's eSIM IoT object sitting between uCIFI allocations.

---

## Object ID Allocation Ranges

Normative source: Core TS 1.2.2 Appendix D.2.1, Table D.2.1-1 "LwM2M Object Classes".

| Range | Class | URN segment | Purpose |
|-------|-------|-------------|---------|
| 0–1023 | OMA | `oma` | Objects produced by the Open Mobile Alliance (Core objects 0–7, companion-enabler objects 8–28, further OMA objects such as 500–510) |
| 1024–2047 | reserved | — | Reserved for future use |
| 2048–10240 | SDO | `ext` | Objects registered by third-party standards organisations or alliances (oneM2M 2048+, IPSO 3200–3350, GSMA/OpenAIS/uCIFI 3351+) |
| 10241–32768 | Vendors | `x` | Objects registered by vendors or individuals (published in the registry) |
| 32769–42768 | Company Reserved | `x` | Company bulk reservation of up to 50 Object IDs for private Objects **not published** on the OMNA registry |
| 42769–42800 | Test | `x` | Testing only; MUST NOT be used in production (collisions possible) |
| 42801–65534 | reserved | — | Reserved for future use |
| 65535 | MAX_ID | — | Reserved; MUST NOT identify an Object (Core TS 1.2.2 Table 7.4-1) |

Observed registrations inside these classes (registry evidence, not spec): IPSO 3200–3350, uCIFI 3410–3449 (with 3441–3443 held by Eclipse and GSMA), SEW Digital Utility 10262–10284. There is no spec-defined "OMNA experimental range 26241–32768"; the only no-registration bands are Company Reserved (32769–42768) and Test (42769–42800). A validator that rejects *custom* objects with `ObjectID < 32769` is applying the Company Reserved boundary as platform policy.

Authoritative allocation rules, URN composition, and validation fault codes are in registry-ddf-authoring.md.

---

## Object Versioning

Objects carry an Object Version (e.g. "1.0", "1.1") independent of the LwM2M Enabler version (Core TS 1.2.2 §7.2).

- Format (§7.2.2): `Major.Minor`. Version 1.0 MAY be omitted from the URN — `urn:oma:lwm2m:oma:44` and `urn:oma:lwm2m:oma:44:1.0` are both valid for v1.0.
- Each new version gets a **new URN** `urn:oma:lwm2m:{oma,ext,x}:ObjectID[:version]` registered at OMNA; the Object ID does not change (§7.2.1).
- Communicated in Register / Update / Discover with the `ver` attribute on the Object link (`</44>;ver=2.2`), or on the single Object Instance link when exactly one instance exists (§7.2.3). The server assumes 1.0 — or the version shipped with the client's Enabler version for core objects — when `ver` is absent.
- **Type I — breaking → Major bump** (§7.2.1): changing the Object's Instances characteristic; changing normative text in the Object description; adding or removing a *mandatory* Resource; modifying a Resource's allowed operations; changing a Resource's Instances characteristic; changing a Resource's Data Type; modifying a Range (other than adding enumeration values); removing an enumeration value; changing a unit; changing normative text in a Resource description.
- **Type II — backwards-compatible → Minor bump** (§7.2.1): changing non-normative Object text; **adding or removing an *optional* Resource**; adding an enumeration value; reusing an enumeration value previously marked "reserved"; changing non-normative Resource text.
- Any other change is forbidden (renaming an Object or Resource, reusing a removed Resource ID).
- Even an optional-resource addition therefore **requires** a new Minor version and URN — e.g. Security 1.1 → 1.2 (R18–30), Connectivity Monitoring 1.2 → 1.3 (R13). Examples of Type I: Event Log 1.0 → 2.0 (Single → Multiple), OSCORE 1.1 → 2.0 (String → Opaque), Gateway 1.0 → 2.0 (Prefix RW → R, R2 removed, R3 added), DevCapMgmt 1.0 → 2.0 (opDisable Multiple → Single).
- The server must handle version differences gracefully; it MUST be able to determine the Object specification a client registers without ambiguity.

---

## Reusable Resources

Resource IDs are 16-bit; each Object specification owns an Object-specific range, and OMA assigns a separate **Reusable Resource ID** range that is re-usable across Objects (Core TS 1.2.2 §7.4 Table 7.4-1, §7.4.2, Appendix D.2.2). Once registered, a Reusable Resource's Name, ID, Operations, Type, Range, Units and Description are frozen; only Instances (Single/Multiple) and Mandatory are decided by the hosting Object (§7.4.2). In the registry's `Common.xml` the reusable IDs occupy 4000–4050 (OMA: ObjectInstanceHandle 4000, Event Log 4010–4017, …) and 5500–6057 (IPSO: 5500 Digital Input State … 5700 Sensor Value … 6000 Interval Period, 6050 Fractional Timestamp).

Common reusable resources:
- 4000: ObjectInstanceHandle (Objlnk)
- 4010–4017: Event Log resources (LogClass … LogDataIncrementalMinimumSendInterval)
- 5500: Digital Input State (Boolean)
- 5501: Digital Input Counter (Integer)
- 5601: Min Measured Value (Float)
- 5602: Max Measured Value (Float)
- 5603: Min Range Value (Float)
- 5604: Max Range Value (Float)
- 5700: Sensor Value (Float) — used by all IPSO sensor objects
- 5701: Sensor Units (String)
- 5750: Application Type (String)
- 5751: Sensor Type (String)

---

## OMNA Registry Process

To register a new object:
1. Create an Issue on the `OpenMobileAlliance/lwm2m-registry` GitHub repository
2. Use the OMA LwM2M Editor to create a valid Object XML file (must validate against `LWM2M.xsd` or `LWM2M-v1_1.xsd`, Core TS 1.2.2 Appendix D)
3. Include a BSD-3 Clause license (required for One Data Model compatibility)
4. Submit a Pull Request with the XML file
5. OMA staff reviews and allocates the Object ID
6. After approval, the object appears in the OMNA registry

Reusable Resources follow a similar process. Object IDs are formally allocated by OMA; do not self-assign from the registered ranges.

**Citation caveat:** Core TS 1.2.2 Appendix E does carry object definitions (E.1 Security … E.14 5GNR Connectivity), so "Core TS 1.2.2 E.6" is a valid citation for the Firmware Update snapshot in that release — but Appendix E is explicitly a snapshot "at the time of the enabler release"; objects evolve independently and the OMNA DDF (ObjectID + ObjectVersion / URN) is the authoritative current definition. Objects 8–20, 22 and 28 have no Core appendix at all — cite their companion TS or DDF. See registry-ddf-authoring.md.

For DDF.xml schema, validation fault codes (413 Type/Operation, 417 SenML Unit, 418 RangeEnumeration, etc.), and the OMNA Standard/Full/custom library model, see registry-ddf-authoring.md.
