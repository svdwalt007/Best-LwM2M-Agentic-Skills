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

Resources have defined types, operations (R/W/RW/E), and optionality (Mandatory/Optional).

---

## Core Objects (0–7)

### Object 0: LwM2M Security

| Resource ID | Name | Type | Operations | Description |
|-------------|------|------|------------|-------------|
| 0 | LwM2M Server URI | String | — | `coaps://host:port` or `coap://host:port` |
| 1 | Bootstrap-Server | Boolean | — | True if this is a bootstrap server |
| 2 | Security Mode | Integer | — | 0=PSK, 1=RPK, 2=x509, 3=NoSec, 4=EST |
| 3 | Public Key or Identity | Opaque | — | PSK identity, RPK, or client certificate |
| 4 | Server Public Key | Opaque | — | Server's public key or trust anchor CA cert |
| 5 | Secret Key | Opaque | — | PSK value or client private key |
| 6 | SMS Security Mode | Integer | — | SMS-specific security (DTLS/3GPP) |
| 7 | SMS Binding Key Parameters | Opaque | — | SMS security key material |
| 8 | SMS Binding Secret Keys | Opaque | — | SMS security secret keys |
| 9 | LwM2M Server SMS Number | String | — | Server's MSISDN for SMS binding |
| 10 | Short Server ID | Integer | — | Links to Server Object (/1) instance |
| 11 | Client Hold Off Time | Integer | — | Bootstrap hold-off (seconds) |
| 12 | BS Account Timeout | Integer | — | Bootstrap account lifetime (seconds) |
| 13 | TLS/DTLS Ciphersuite | Integer | — | Negotiated cipher suite ID (v1.2+) |
| 14 | DTLS/TLS Version | String | — | Min/max version constraints (v1.2+) |
| 15 | OSCORE Security Mode | Objlnk | — | Link to OSCORE Object /21 instance (v1.1+) |
| 16 | Certificate Usage | Integer | — | X.509 certificate usage type (v1.2+) |
| 17 | TLS-DTLS Alert Code | Integer | R | Last TLS/DTLS alert code (v1.2+) |
| 18 | SNI | String | — | Server Name Indication value (v1.2+) |
| 19 | Certificate | Objlnk | — | Link to certificate object (v1.2+) |

**Notes:** Security Object instances are not accessible via the DM&SE interface. They are provisioned only through the Bootstrap Interface or factory provisioning. Each instance maps to one server relationship.

### Object 1: LwM2M Server

| Resource ID | Name | Type | Operations | Description |
|-------------|------|------|------------|-------------|
| 0 | Short Server ID | Integer | R | Unique ID for this server |
| 1 | Lifetime | Integer | RW | Registration lifetime in seconds |
| 2 | Default Minimum Period | Integer | RW | Default pmin for observations |
| 3 | Default Maximum Period | Integer | RW | Default pmax for observations |
| 4 | Disable | — | E | Disable this server relationship |
| 5 | Disable Timeout | Integer | RW | Auto-re-enable after N seconds |
| 6 | Notification Storing When Disabled/Offline | Boolean | RW | Buffer notifications during sleep |
| 7 | Binding | String | RW | Transport binding: U, UQ, T, TQ, S, etc. |
| 8 | Registration Update Trigger | — | E | Force registration update |
| 9 | Bootstrap-Request Trigger | — | E | Force bootstrap request (v1.1+) |
| 10 | APN Link | Objlnk | RW | Link to APN Connection Profile /11 (v1.1+) |
| 11 | TLS-DTLS Alert Code | Integer | R | Last alert code (v1.1+) |
| 12 | Last Bootstrapped | Time | R | Timestamp of last bootstrap (v1.1+) |
| 23 | Mute Send | Boolean | RW | Disable Send operation for this server (v1.2+) |
| 24 | Alternative Content Formats | Integer | R | Supported non-default formats (v1.2+) |

### Object 2: Access Control
Multi-instance. One instance per Object Instance accessible by multiple servers.

| Resource ID | Name | Type | Operations |
|-------------|------|------|------------|
| 0 | Object ID | Integer | R |
| 1 | Object Instance ID | Integer | R |
| 2 | ACL | Integer | RW (multi-instance) |
| 3 | Access Control Owner | Integer | RW |

ACL bits: 0=none, 1=Read, 2=Write, 4=Execute, 8=Create, 16=Delete. Multiple operations combine (e.g., 3 = Read+Write).

### Object 3: Device
Single instance. Mandatory for all LwM2M clients.

| Resource ID | Name | Type | Operations |
|-------------|------|------|------------|
| 0 | Manufacturer | String | R |
| 1 | Model Number | String | R |
| 2 | Serial Number | String | R |
| 3 | Firmware Version | String | R |
| 4 | Reboot | — | E |
| 5 | Factory Reset | — | E |
| 6 | Available Power Sources | Integer | R (multi) |
| 7 | Power Source Voltage | Integer | R (multi) |
| 8 | Power Source Current | Integer | R (multi) |
| 9 | Battery Level | Integer | R |
| 10 | Memory Free | Integer | R |
| 11 | Error Code | Integer | R (multi) |
| 12 | Reset Error Code | — | E |
| 13 | Current Time | Time | RW |
| 14 | UTC Offset | String | RW |
| 15 | Timezone | String | RW |
| 16 | Supported Binding & Modes | String | R |
| 17 | Device Type | String | R |
| 18 | Hardware Version | String | R |
| 19 | Software Version | String | R |
| 20 | Battery Status | Integer | R |
| 21 | Memory Total | Integer | R |
| 22 | ExtDevInfo | Objlnk | R (multi) |

**Mandatory resources:** Reboot (/3/0/4) is mandatory; Error Code (/3/0/11) and Supported Binding & Modes (/3/0/16) are mandatory. The remaining resources are optional. Current Time (/3/0/13), UTC Offset (/3/0/14), and Timezone (/3/0/15) are RW Time/String resources; Reboot (/3/0/4), Factory Reset (/3/0/5), and Reset Error Code (/3/0/12) are Execute-only.

### Object 4: Connectivity Monitoring
Single instance. Network connectivity status.

Key resources: Network Bearer (0), Available Network Bearer (1), Radio Signal Strength (2), Link Quality (3), IP Addresses (4, multi), Router IP Addresses (5, multi), Link Utilisation (6), APN (7, multi), Cell ID (8), SMNC (9), SMCC (10), SignalSNR (11), LAC (12).

### Object 5: Firmware Update
Single instance. Manages the firmware update state machine.

| Resource ID | Name | Type | Operations |
|-------------|------|------|------------|
| 0 | Package | Opaque | W |
| 1 | Package URI | String | W |
| 2 | Update | — | E |
| 3 | State | Integer | R |
| 5 | Update Result | Integer | R |
| 6 | PkgName | String | R |
| 7 | PkgVersion | String | R |
| 8 | Firmware Update Protocol Support | Integer | R (multi) |
| 9 | Firmware Update Delivery Method | Integer | R |

**State machine:** 0=Idle → 1=Downloading → 2=Downloaded → 3=Updating → 0=Idle (success) or 0=Idle (fail with Update Result code).

**Delivery methods:** 0=Pull only, 1=Push only, 2=Both.

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
| 0 | Initial (no firmware update yet, or after a successful reset) |
| 1 | Firmware updated successfully |
| 2 | Not enough flash |
| 3 | Out of RAM |
| 4 | Connection lost during download |
| 5 | Integrity check failure |
| 6 | Unsupported package type |
| 7 | Invalid URI |
| 8 | Firmware update failed |
| 9 | Unsupported protocol |

**Delivery Method (/5/0/9):** 0=Pull, 1=Push, 2=Both.
- **PULL** — the server writes the Package URI to /5/0/1 and the client fetches the binary itself.
- **PUSH** — the server writes the binary directly to /5/0/0 Package (Opaque).

A conformant client sets **Update Result** (not just State) on every terminal outcome, so the server can distinguish success from each failure cause.

### Object 6: Location
Single instance. GPS/GNSS data.

Key resources: Latitude (0), Longitude (1), Altitude (2), Radius (3), Velocity (4), Timestamp (5), Speed (6).

### Object 7: Connectivity Statistics
Single instance. Data usage tracking.

Key resources: SMS TX Counter (0), SMS RX Counter (1), TX Data (2), RX Data (3), Max Message Size (4), Average Message Size (5), Collection Period (6, RW — start/stop with Write).

---

## Extended OMA object resource maps

### Object 9: Software Management

Multi-instance — one instance per software bundle. Distinct from /5, which owns the firmware/OS image; /9 manages installable application software packages.

| Resource ID | Name | Type | Operations | Description |
|-------------|------|------|------------|-------------|
| 0 | PkgName | String | R | Name of the software package |
| 1 | PkgVersion | String | R | Version of the software package |
| 2 | Package | Opaque | W | Software binary delivered by push |
| 3 | PackageURI | String | RW | URI the client fetches the package from (pull) |
| 4 | Install | — | E | Install the downloaded package |
| 5 | Checkpoint | Objlnk | R | Link to a checkpoint object |
| 6 | Uninstall | — | E | Uninstall the package |
| 7 | Update State | Integer | R | Software update state machine |
| 8 | Update Supported Objects | Boolean | RW | Whether supported objects are updated on install |
| 9 | Update Result | Integer | R | Result code of the last install/uninstall |
| 10 | Activate | — | E | Activate the installed software |
| 11 | Deactivate | — | E | Deactivate the installed software |
| 12 | Activation State | Boolean | R | True if the software is currently active |
| 13 | Package Settings | Objlnk | RW | Link to package-specific settings object |

### Object 11: APN Connection Profile

Multi-instance — one instance per APN profile. Linked from the Server Object via /1/x/10 APN Link.

| Resource ID | Name | Type | Operations | Description |
|-------------|------|------|------------|-------------|
| 0 | Profile Name | String | RW | Human-readable profile name (mandatory) |
| 1 | APN | String | RW | Access Point Name |
| 2 | Auto Select APN by Device | Boolean | RW | Device selects APN automatically |
| 3 | Enable Status | Boolean | RW | Whether this profile is enabled |
| 4 | Authentication Type | Integer | RW | Auth type enum (mandatory): 0=PAP, 1=CHAP, 2=PAP or CHAP, 3=None |
| 5 | User Name | String | RW | Authentication user name |
| 6 | Secret | String | W | Authentication secret — **Write-only** so a server cannot read the credential back |
| 7 | Reconnect Schedule | String | RW | Schedule for reconnection attempts |
| 8 | Validity (MCC-MNC) | String | R (multi) | PLMNs for which this profile is valid |
| 9 | Connection Establishment Time | Time | R (multi) | Timestamp of connection establishment |
| 10 | Connection Establishment Result | Integer | R (multi) | Result of connection establishment |
| 11 | Connection End Time | Time | R (multi) | Timestamp of connection teardown |
| 12 | Total Bytes Sent | Integer | R | Cumulative bytes transmitted |
| 13 | Total Bytes Received | Integer | R | Cumulative bytes received |
| 14 | IP Address | String | R (multi) | Assigned IP address(es) |

**Note:** Credential resources such as Secret (/11/0/6) are exposed Write-only so a server can provision but never retrieve them.

### Object 25: LwM2M Gateway

Multi-instance — one instance per aggregated non-LwM2M end-device. An OMA Technical Specification object (OMA-TS-LwM2M_Gateway), not part of the core Enabler set. This is the canonical gateway/aggregation pattern for representing non-LwM2M devices behind a single LwM2M client.

| Resource ID | Name | Type | Operations | Description |
|-------------|------|------|------------|-------------|
| 0 | Device ID | String | R | Identifier of the bridged end-device (mandatory) |
| 1 | Prefix | String | RW | URI path segment under which the server addresses the bridged device's objects as sub-endpoints (mandatory) |
| 2 | IoT Device Objects | String | R | CoreLink string of the device's object tree, e.g. `</3>,</4>` (mandatory) |
| 3 | Device Status | String | R | Status of the bridged end-device |

---

## Extended OMA Objects (8–28)

| Object ID | Name | Version Added | Key Purpose |
|-----------|------|---------------|-------------|
| 8 | Lock and Wipe | v1.0 | Remote device lock/wipe for theft protection |
| 9 | Software Management | v1.0 | Install/uninstall software packages |
| 10 | Cellular Connectivity | v1.1 | Cellular radio configuration (APN, roaming) |
| 11 | APN Connection Profile | v1.1 | Per-APN configuration (authentication, PDN type) |
| 12 | WLAN Connectivity | v1.1 | Wi-Fi network configuration |
| 13 | Bearer Selection | v1.1 | Bearer preference rules |
| 14 | Software Component | v1.1 | Software component inventory |
| 15 | DevCapMgmt | v1.1 | Device capability management |
| 16 | Portfolio | v1.1 | Software/firmware portfolio tracking |
| 17 | Communications Characteristics | v1.0 | Communication parameters / characteristics configuration |
| 18 | Non-Access Stratum (NAS) Configuration | v1.0 | 3GPP NAS configuration parameters |
| 19 | BinaryAppDataContainer | v1.1 | Generic binary application data exchange |
| 20 | Event Log | v1.1 | Device event/log management |
| 21 | OSCORE | v1.2 | OSCORE security context parameters |
| 22 | Virtual Observe Notify | v1.1 | Observe/notify multiple resources across objects/instances in fewer messages |
| 23 | LwM2M COSE | v1.2 | COSE key/credential parameters (used with the MQTT transport binding) |
| 24 | MQTT Server | v1.2 | MQTT broker connection configuration (MQTT transport binding) |
| 25 | LwM2M Gateway | Gateway TS | Smart proxy: holds one instance per connected downstream IoT device, mapping object paths and routing commands (e.g. CoAP GET) so non-LwM2M / constrained devices are managed by the central server. See the resource map under "Object 25: LwM2M Gateway" above |
| 26 | LwM2M Gateway Routing | Gateway TS | Routing table for devices proxied behind a LwM2M Gateway (/25) |
| 27 | 5GNR Connectivity | v1.2 | 5G NR connectivity parameters |
| 28 | Device RF Capabilities | v1.2 | Device radio-frequency capability reporting |

> **Source of truth:** the IDs/names above are reconciled against the OMNA LwM2M Object registry (`openmobilealliance.org/specifications/registries/objects`, mirrored at `OpenMobileAlliance/lwm2m-registry` → `prod/DDF.xml`). Object **23 = LwM2M COSE**, **24 = MQTT Server**, **25 = LwM2M Gateway**, **26 = LwM2M Gateway Routing** — confirmed against the registry and the OMA MQTT-binding / Gateway / Virtual Observe Notify specifications. (An earlier revision had the 22–26 block scrambled; there is **no** "reserved/unassigned" gap at 23.) **Version Added** reflects best-known introduction — verify each against the object's DDF `ObjectVersion`; note that **Objects 25 and 26 (Gateway, Gateway Routing) are defined in the separate OMA LwM2M Gateway Technical Specification**, not the core Enabler.

---

## IPSO Smart Objects

After the IPSO Alliance merged with OMA in 2018, IPSO Smart Objects became part of the OMNA LwM2M Registry. These are generic sensor/actuator objects in the 3300–3400+ range:

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

Selected notable third-party registrations:

| Object ID Range | Organisation | Domain |
|-----------------|-------------|--------|
| 500-509 | OMA / 3GPP | eSIM provisioning, RSP (TODO(verify)) |
| 2048-2049 | GSMA | CIoT device management (TODO(verify)) |
| 3200-3499 | IPSO | IPSO Smart Objects |
| 3400-3449 | uCIFI | Smart City (carved out of the IPSO band) |
| 10262-10284 | SEW | Digital Utility — water metering |
| 10300-10399 | Various vendors | Vendor device management (TODO(verify)) |
| 10350-10375 | Industrial IoT | Modbus integration, BACnet (TODO(verify)) |

---

## Object ID Allocation Ranges

| Range | Purpose |
|-------|---------|
| 0–7 | OMA core objects |
| 8–42 | OMA extended objects (TODO(verify)) |
| 43–1023 | Reserved for OMA and external SDOs (TODO(verify)) |
| 1024–2047 | Registered external SDO objects (TODO(verify)) |
| 2048–10239 | Registered industry objects (TODO(verify)) |
| 3200–3499 | IPSO Smart Objects (uCIFI Smart City carved out as 3400–3449) |
| 10240–26240 | Registered vendor objects (TODO(verify)) |
| 10262–10284 | SEW Digital Utility — water metering |
| 26241–32768 | OMNA experimental range (private/test, no registration required) |
| 32769+ | Vendor-private IDs (validation rejects ObjectID < 32768) |

Authoritative allocation rules, URN composition, and validation fault codes are in registry-ddf-authoring.md.

---

## Object Versioning

Objects carry a version number (e.g., "1.0", "1.1") independent of the LwM2M protocol version.

- Object version is communicated in registration: `</objectID>;ver=X.Y`
- The version defaults to "1.0" if not specified
- A new version is required when: mandatory resources are added, resource semantics change, or resource types change
- Optional resource additions do not require a version bump
- The server must handle version differences gracefully

---

## Reusable Resources

Resources with IDs in the range 4000–32768 are "reusable" — defined once in the OMNA registry and shareable across multiple objects. This avoids duplicate definitions.

Common reusable resources:
- 4000: ObjectInstanceHandle (Objlnk)
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
2. Use the OMA LwM2M Editor to create a valid Object XML file
3. Include a BSD-3 Clause license (required for One Data Model compatibility)
4. Submit a Pull Request with the XML file
5. OMA staff reviews and allocates the Object ID
6. After approval, the object appears in the OMNA registry

Reusable Resources follow a similar process. Object IDs are formally allocated by OMA; do not self-assign from the registered ranges.

**Citation caveat:** Do not cite objects by an invented spec-annex letter (e.g. '§E.6'); cite the OMNA Object ID/URN and the object's DDF ObjectVersion. See registry-ddf-authoring.md.

For DDF.xml schema, validation fault codes (413 Type/Operation, 417 SenML Unit, 418 RangeEnumeration, etc.), and the OMNA Standard/Full/custom library model, see registry-ddf-authoring.md.
