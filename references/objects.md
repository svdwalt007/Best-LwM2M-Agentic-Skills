# OMA LwM2M Objects — Reference

## Table of Contents
1. [Object Model Overview](#object-model-overview)
2. [Core Objects (0–7)](#core-objects-07)
3. [Extended OMA Objects (8–25)](#extended-oma-objects-825)
4. [IPSO Smart Objects](#ipso-smart-objects)
5. [Industry-Registered Objects](#industry-registered-objects)
6. [Object ID Allocation Ranges](#object-id-allocation-ranges)
7. [Object Versioning](#object-versioning)
8. [Reusable Resources](#reusable-resources)
9. [OMNA Registry Process](#omna-registry-process)

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

### Object 6: Location
Single instance. GPS/GNSS data.

Key resources: Latitude (0), Longitude (1), Altitude (2), Radius (3), Velocity (4), Timestamp (5), Speed (6).

### Object 7: Connectivity Statistics
Single instance. Data usage tracking.

Key resources: SMS TX Counter (0), SMS RX Counter (1), TX Data (2), RX Data (3), Max Message Size (4), Average Message Size (5), Collection Period (6, RW — start/stop with Write).

---

## Extended OMA Objects (8–25)

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
| 18 | Non-IP Data Delivery (NIDD) | v1.1 | 3GPP Non-IP Data Delivery configuration |
| 19 | BinaryAppDataContainer | v1.1 | Generic binary application data exchange |
| 20 | Event Log | v1.1 | Device event/log management |
| 21 | OSCORE | v1.2 | OSCORE security context parameters |
| 22 | MQTT Server | v1.2 | MQTT broker connection configuration |
| 23 | LwM2M Gateway | v1.2 | Gateway device registry and management |
| 24 | LwM2M Gateway Routing | v1.2 | Gateway routing table for proxied devices |
| 25 | LwM2M COSE | v1.2 | COSE key and credential management |

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
| 500-509 | OMA / 3GPP | eSIM provisioning, RSP |
| 2048-2049 | GSMA | CIoT device management |
| 3200-3203 | uCIFI | Smart City sensors |
| 3400-3443 | 3GPP / OMA | Connectivity management, eSIM |
| 10241-10299 | AVSystem | Anjay-specific extensions |
| 10300-10399 | Various vendors | Vendor device management |
| 10350-10375 | Industrial IoT | Modbus integration, BACnet |

---

## Object ID Allocation Ranges

| Range | Purpose |
|-------|---------|
| 0–7 | OMA core objects |
| 8–42 | OMA extended objects |
| 43–1023 | Reserved for OMA and external SDOs |
| 1024–2047 | Registered external SDO objects |
| 2048–10239 | Registered industry objects |
| 10240–26240 | Registered vendor objects |
| 26241–32768 | Private/test objects (no registration required) |
| 33000+ | Vendor-specific (no registration required) |

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
