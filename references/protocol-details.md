# LwM2M Protocol Details — CoAP, Transport Bindings & Message Flows

## Table of Contents
1. [CoAP Quick Reference](#coap-quick-reference)
2. [LwM2M Interface to CoAP Mapping](#lwm2m-interface-to-coap-mapping)
3. [Registration Flow](#registration-flow)
4. [Bootstrap Flows](#bootstrap-flows)
5. [Observation & Notification](#observation--notification)
6. [Composite Operations (v1.2+)](#composite-operations-v12)
7. [Send Operation (v1.2+)](#send-operation-v12)
8. [Queue Mode](#queue-mode)
9. [Blockwise Transfer](#blockwise-transfer)
10. [Content-Format Negotiation](#content-format-negotiation)
11. [Transport Binding Details](#transport-binding-details)
12. [Wireshark/tshark Analysis Patterns](#wiresharktshark-analysis-patterns)

---

## CoAP Quick Reference

LwM2M uses CoAP (RFC 7252) as its application protocol. Key CoAP concepts:

**Message Types:**
- CON (Confirmable) — requires ACK, reliable delivery
- NON (Non-Confirmable) — fire-and-forget, unreliable
- ACK (Acknowledgement) — response to CON
- RST (Reset) — indicates message rejection

**Response Codes (most relevant to LwM2M):**
| Code | Meaning | LwM2M Usage |
|------|---------|-------------|
| 2.01 | Created | Register success, Create success |
| 2.02 | Deleted | De-register success, Delete success |
| 2.04 | Changed | Update success, Write success, Execute success |
| 2.05 | Content | Read response, Observe/Notify response |
| 2.31 | Continue | Blockwise transfer continuation |
| 4.00 | Bad Request | Malformed request |
| 4.01 | Unauthorized | Failed authentication |
| 4.03 | Forbidden | No access rights (ACL) |
| 4.04 | Not Found | Object/Instance/Resource does not exist |
| 4.05 | Method Not Allowed | Operation not supported (e.g., Execute on R-only) |
| 4.06 | Not Acceptable | Content-format not supported |
| 4.08 | Request Entity Incomplete | Missing block |
| 4.12 | Precondition Failed | Conditional request failed |
| 4.15 | Unsupported Content-Format | Payload format not understood |
| 5.00 | Internal Server Error | Implementation failure |
| 5.03 | Service Unavailable | Temporarily unable to handle |

**Key CoAP Options for LwM2M:**
| Option | Number | Purpose in LwM2M |
|--------|--------|-------------------|
| Uri-Path | 11 | Resource addressing (/rd, /bs, /dp, object paths) |
| Content-Format | 12 | TLV(11542), JSON(11543), SenML-CBOR(112), LwM2M-CBOR |
| Accept | 17 | Client's preferred response format |
| Uri-Query | 15 | Registration params: ep, lt, lwm2m, b, sms |
| Observe | 6 | 0=Register observation, 1=Deregister |
| Block1 | 27 | Blockwise request body (firmware upload) |
| Block2 | 23 | Blockwise response body |
| Size1 | 60 | Request body size hint |
| Size2 | 28 | Response body size hint |
| ETag | 4 | Cache validation |
| Location-Path | 8 | Registration location: /rd/{registration-id} |

---

## LwM2M Interface to CoAP Mapping

### Registration Interface
| Operation | CoAP Method | URI | Key Options |
|-----------|------------|-----|-------------|
| Register | POST | /rd | Uri-Query: ep, lt, lwm2m, b |
| Update | POST | /rd/{id} | Uri-Query: lt, b (if changed) |
| De-register | DELETE | /rd/{id} | — |

### Device Management & Service Enablement Interface
| Operation | CoAP Method | URI | Content-Format |
|-----------|------------|-----|----------------|
| Read | GET | /{O}/{I} or /{O}/{I}/{R} | Accept header |
| Discover | GET | /{O} or /{O}/{I} or /{O}/{I}/{R} | Accept: application/link-format |
| Write | PUT | /{O}/{I}/{R} or /{O}/{I} | TLV/SenML/CBOR |
| Write (partial) | POST | /{O}/{I} | TLV/SenML/CBOR |
| Execute | POST | /{O}/{I}/{R} | text/plain (arguments) |
| Create | POST | /{O} | TLV/SenML/CBOR |
| Delete | DELETE | /{O}/{I} | — |
| Write-Attributes | PUT | /{O} or /{O}/{I} or /{O}/{I}/{R} | Uri-Query: pmin, pmax, gt, lt, st |
| Read-Composite | FETCH | / | SenML/LwM2M-CBOR |
| Write-Composite | iPATCH | / | SenML/LwM2M-CBOR |

### Information Reporting Interface
| Operation | CoAP Method | URI |
|-----------|------------|-----|
| Observe | GET + Observe:0 | /{O}/{I}/{R} |
| Cancel Observation | GET + Observe:1 or RST | /{O}/{I}/{R} |
| Notify | (async response) | — |
| Send | POST | /dp | SenML/LwM2M-CBOR |
| Observe-Composite | FETCH + Observe:0 | / |

---

## Registration Flow

```
Client                                    Server
  │                                          │
  │── POST /rd?ep=myDevice                ──►│
  │   &lt=300&lwm2m=1.2&b=UQ                  │
  │   Content-Format: application/link-format│
  │   Payload: </0/0>,</0/1>,</1/0>,         │
  │            </3/0>,</4/0>,</5/0>          │
  │                                          │
  │◄── 2.01 Created ────────────────────────│
  │    Location-Path: /rd/a1b2c3             │
  │                                          │
  │    ... time passes (before lt expires)   │
  │                                          │
  │── POST /rd/a1b2c3 ─────────────────────►│
  │   (Registration Update)                  │
  │                                          │
  │◄── 2.04 Changed ───────────────────────│
  │                                          │
  │    ... when shutting down ...             │
  │                                          │
  │── DELETE /rd/a1b2c3 ───────────────────►│
  │   (De-register)                          │
  │                                          │
  │◄── 2.02 Deleted ───────────────────────│
```

The registration payload lists all Object Instances the client supports, in CoRE Link Format. Object versions are included via the `ver` attribute when non-default.

---

## Bootstrap Flows

### Client-Initiated Bootstrap
```
Client                              Bootstrap-Server
  │                                       │
  │── POST /bs?ep=myDevice ─────────────►│
  │                                       │
  │◄── 2.04 Changed ───────────────────  │
  │                                       │
  │◄── DELETE / ────────────────────────  │  (clean slate)
  │── 2.02 Deleted ────────────────────► │
  │                                       │
  │◄── PUT /0/0 (Security for BS) ─────  │  (write security objects)
  │── 2.04 Changed ────────────────────► │
  │                                       │
  │◄── PUT /0/1 (Security for Server) ─  │
  │── 2.04 Changed ────────────────────► │
  │                                       │
  │◄── PUT /1/0 (Server Object) ───────  │
  │── 2.04 Changed ────────────────────► │
  │                                       │
  │◄── POST /bs (Bootstrap-Finish) ────  │
  │── 2.04 Changed ────────────────────► │
  │                                       │
  │   [Client now registers with Server]  │
```

### Bootstrap-Pack-Request (v1.2+)
```
Client                              Bootstrap-Server
  │                                       │
  │── POST /bspack?ep=myDevice ────────►│
  │                                       │
  │◄── 2.05 Content ──────────────────  │
  │    Payload: complete bootstrap config │
  │    (Security + Server objects in one  │
  │     SenML-CBOR or LwM2M-CBOR payload)│
```

Reduces the bootstrap exchange from many round-trips to a single request-response.

---

## Observation & Notification

### Observation Attributes
| Attribute | Meaning | Scope |
|-----------|---------|-------|
| pmin | Minimum period between notifications (seconds) | Object/Instance/Resource |
| pmax | Maximum period between notifications (seconds) | Object/Instance/Resource |
| gt | Greater Than — notify when value exceeds threshold | Resource |
| lt | Less Than — notify when value drops below threshold | Resource |
| st | Step — notify on change exceeding this magnitude | Resource |
| epmin | Minimum evaluation period (v1.2+) | Resource |
| epmax | Maximum evaluation period (v1.2+) | Resource |
| edge | Edge notification — trigger on boolean threshold crossing (v1.2+) | Resource |
| con | Confirmable notification — use CON messages (v1.2+) | Object/Instance/Resource |
| hqmax | Maximum Historical Queue — buffer up to N notifications (v1.2+) | Object/Instance/Resource |

### Notification Conditions (per OMA-TS-LightweightM2M_Core §6.4.2)
Notifications are sent when ALL conditions are met:
1. pmin timer has expired (minimum time between notifications)
2. At least one value condition is met (gt, lt, st, or edge triggered)
3. OR pmax timer expires (maximum silence period)

---

## Composite Operations (v1.2+)

### Read-Composite
Uses the CoAP FETCH method (RFC 8132). Payload contains a list of paths to read.

```
Client                          Server
  │                               │
  │◄── FETCH / ─────────────────  │
  │    Content-Format: SenML-CBOR │
  │    Payload: ["/3/0/0",        │
  │              "/3/0/9",        │
  │              "/4/0/2"]        │
  │                               │
  │── 2.05 Content ────────────► │
  │   Content-Format: SenML-CBOR  │
  │   Payload: [{bn:"/3/0/",      │
  │              n:"0", vs:"Mfr"},│
  │             {n:"9", v:85},    │
  │             {bn:"/4/0/",      │
  │              n:"2", v:-72}]   │
```

### Write-Composite
Uses CoAP iPATCH (RFC 8132). Payload contains path-value pairs to write.

### Observe-Composite
Uses FETCH + Observe:0. The client receives notifications whenever any of the observed paths changes.

---

## Send Operation (v1.2+)

Device-initiated data push. The client sends data to the server without a prior request.

```
Client                          Server
  │                               │
  │── POST /dp ────────────────► │
  │   Content-Format: SenML-CBOR  │
  │   Payload: [{bn:"/3303/0/",   │
  │              n:"5700",v:23.5}, │
  │             {bn:"/3304/0/",   │
  │              n:"5700",v:65}]  │
  │                               │
  │◄── 2.04 Changed ────────────  │
```

The server can mute Send from a specific client via Server Object /1/x/23 (Mute Send = true).

---

## Queue Mode

Queue Mode (binding suffix "Q") is essential for sleepy devices (NB-IoT PSM, LTE-M eDRX).

**Behaviour:**
1. Client registers with binding "UQ" (UDP + Queue Mode)
2. Client enters sleep (radio off / PSM)
3. Server queues any pending operations (Read, Write, Execute, Observe setup)
4. Client wakes and sends Registration Update
5. Server detects client is awake and sends queued operations
6. Client processes operations, then sleeps again

**With DTLS CID (RFC 9146):**
After PSM sleep and NAT rebinding, the client's source IP/port changes. Without CID, this breaks the DTLS session and forces a full re-handshake. With CID, the server matches the CID in the DTLS record header to the existing session — no re-handshake needed.

---

## Blockwise Transfer

For payloads exceeding the CoAP message size (typically ~1024 bytes for UDP), RFC 7959 blockwise transfer is used.

**Firmware Update (Pull method, Block2):**
```
Client                          File Server
  │                               │
  │── GET /firmware.bin ────────► │
  │   Block2: 0/0/1024            │ (request block 0, 1024 byte blocks)
  │                               │
  │◄── 2.05 Content ────────────  │
  │    Block2: 0/1/1024           │ (block 0, more blocks follow)
  │    Payload: [1024 bytes]      │
  │                               │
  │── GET /firmware.bin ────────► │
  │   Block2: 1/0/1024            │
  │                               │
  │◄── 2.05 Content ────────────  │
  │    Block2: 1/1/1024           │
  │    ... continues ...           │
```

**Firmware Update (Push method, Block1):**
Server pushes firmware to client via Write to /5/0/0 (Package resource) using Block1.

---

## Content-Format Negotiation

The server indicates its preferred response format using the CoAP Accept option. The client indicates its supported formats during registration (via the content-format link attribute or through version negotiation).

**Negotiation rules:**
1. If Accept option is present, server requests that format
2. If client can't produce the requested format, it responds 4.06 Not Acceptable
3. Default format depends on version: TLV for v1.0, implementation-dependent for v1.1+
4. SenML-JSON and SenML-CBOR require v1.1+
5. LwM2M CBOR requires v1.2+
6. Composite operations require SenML or LwM2M CBOR formats

---

## Transport Binding Details

### CoAP/UDP (Binding "U")
- Default transport, supported since v1.0
- Security: DTLS 1.2 (RFC 6347), DTLS 1.3 (RFC 9147, v1.2+ optional)
- DTLS CID (RFC 9146) for NAT traversal (v1.2+)
- Block size typically 1024 bytes (configurable)
- Observe notifications via NON or CON messages

### CoAP/TCP (Binding "T")
- Added in v1.1 (RFC 8323)
- Security: TLS 1.2 or TLS 1.3
- No message layer (CON/NON/ACK/RST) — all messages are reliable
- No Block options needed — TCP handles segmentation
- Better for always-connected devices behind NATs/firewalls
- Ping/Pong for keepalive

### CoAP/SMS (Binding "S")
- Supported since v1.0
- SMS as transport for CoAP messages
- Very limited payload (140-160 bytes per SMS)
- High latency, high cost per message
- Useful for SMS-triggered wake-up with main data over UDP

### CoAP over Non-IP (Binding "N")
- Added in v1.1
- 3GPP CIoT: NB-IoT/LTE-M Data-over-NAS and User Plane CIoT optimisation
- LoRaWAN: CoAP mapping over LoRa frames
- Typically very constrained payload sizes (50-240 bytes)
- OSCORE often preferred over DTLS for Non-IP (avoids DTLS handshake overhead)

### MQTT (Binding "M")
- Added in v1.2
- LwM2M messages mapped to MQTT pub/sub topics
- Requires MQTT Server Object /22 configuration
- Useful for cloud platform integration (AWS IoT, Azure IoT Hub)
- Security via TLS on MQTT connection

### HTTP (Binding "H")
- Added in v1.2
- LwM2M messages mapped to HTTP methods
- Security via TLS (HTTPS)
- Useful for gateway/proxy scenarios and enterprise integration

---

## Wireshark/tshark Analysis Patterns

### DTLS CID Field Extraction
```bash
tshark -r capture.pcap -T fields \
  -e frame.number -e ip.src -e ip.dst \
  -e udp.srcport -e udp.dstport \
  -e dtls.record.content_type \
  -e dtls.record.epoch \
  -e dtls.record.sequence_number \
  -e dtls.handshake.type \
  -e dtls.handshake.extensions.connection_id \
  -E header=y -E separator='|'
```

### Filter for CID-bearing Records (ContentType 25)
```bash
tshark -r capture.pcap -Y "dtls.record.content_type == 25"
```

### LwM2M Registration Tracking
```bash
tshark -r capture.pcap -T fields \
  -e frame.number -e coap.code \
  -e coap.opt.uri_path -e coap.opt.uri_query \
  -Y "coap" -E header=y -E separator='|'
```

### Filter for Specific LwM2M Operations
```bash
# Registration: POST to /rd
tshark -r capture.pcap -Y 'coap.code == 2 && coap.opt.uri_path == "rd"'

# Bootstrap Request: POST to /bs
tshark -r capture.pcap -Y 'coap.code == 2 && coap.opt.uri_path == "bs"'

# Send operation: POST to /dp
tshark -r capture.pcap -Y 'coap.code == 2 && coap.opt.uri_path == "dp"'

# Observe notifications
tshark -r capture.pcap -Y "coap.opt.observe"
```

### Decode TLV Payload
```bash
tshark -r capture.pcap -V -Y "frame.number==42" | \
  grep -A 50 "Payload"
```
