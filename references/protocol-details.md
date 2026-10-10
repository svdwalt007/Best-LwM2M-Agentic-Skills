# LwM2M Protocol Details — CoAP, Transport Bindings & Message Flows

Normative sources: OMA-TS-LightweightM2M_Core-V1_2_2-20240613-A ("Core TS 1.2.2") and
OMA-TS-LightweightM2M_Transport-V1_2_2-20240613-A ("Transport TS 1.2.2"). Version attribution
("v1.1+", "v1.2") is taken from the corresponding older approved TS.

## Table of Contents
1. [CoAP Quick Reference](#coap-quick-reference)
2. [LwM2M Interface to CoAP Mapping](#lwm2m-interface-to-coap-mapping)
3. [Registration Flow](#registration-flow)
4. [Bootstrap Flows](#bootstrap-flows)
5. [Observation & Notification](#observation--notification)
6. [Composite Operations (v1.1+)](#composite-operations-v11)
7. [Send Operation (v1.1+)](#send-operation-v11)
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

Transport TS 1.2.2 §6.8.1: over UDP every LwM2M operation MUST be a Confirmable CoAP message,
except Notify, Execute and Send, which MAY be Non-Confirmable.

**Response Codes (as used by LwM2M — Transport TS 1.2.2 §6.7, Tables 6.7-1 to 6.7-4):**
| Code | Meaning | LwM2M Usage |
|------|---------|-------------|
| 2.01 | Created | Register success, Create success |
| 2.02 | Deleted | De-register success, Delete success, Bootstrap-Delete success |
| 2.04 | Changed | Update, Write, Write-Composite, Write-Attributes, Execute, Send, Bootstrap-Request, Bootstrap-Finish success |
| 2.05 | Content | Read, Read-Composite, Discover, Observe, Notify, Bootstrap-Pack-Request success |
| 2.31 | Continue | Blockwise Write continuation (Block1) |
| 4.00 | Bad Request | Malformed request; mandatory Register parameter missing or unknown parameter; unknown endpoint name; ep does not match certificate CN; Write data format/type mismatch; Execute argument not understood; inconsistent Write-Attributes |
| 4.01 | Unauthorized | **Access Right Permission Denied** — ACL denial on every DM&SE and Information Reporting operation (Read, Write, Execute, Create, Delete, Discover, Write-Attributes, composites, Observe). On Register: client not authorised; under OSCORE, server rejecting an initial request |
| 4.03 | Forbidden | **Register only**: registration of this Endpoint Client Name is not allowed on this server |
| 4.04 | Not Found | Target Object/Instance/Resource/Resource Instance does not exist; Send: reported Object not registered; Update/De-register on unknown location |
| 4.05 | Method Not Allowed | Operation not supported by target (Execute on a non-executable resource, Write on R-only, Read on a non-readable resource, path-depth rule violated); Bootstrap-Pack-Request declined by the Bootstrap-Server |
| 4.06 | Not Acceptable | Client cannot return any of the preferred Content-Formats (Read/Observe); Content-Format not supported (Write/Create); Bootstrap-Finish: inconsistent loaded configuration |
| 4.08 | Request Entity Incomplete | Blockwise Write: missing block |
| 4.09 | Conflict | **Register only**: server cannot determine the client's Object/Instance list (e.g. unresolved `pid`, or LoRaWAN registration without an object list) |
| 4.12 | Precondition Failed | **Register only**: LwM2M versions of server and client are not compatible |
| 4.13 | Request Entity Too Large | Blockwise Write payload too large |
| 4.15 | Unsupported Content-Format | Payload format not supported (Write, Create, Read-Composite, Observe-Composite, Bootstrap-Request/Write) |
| 5.01 | Not Implemented | Bootstrap-Server does not implement Bootstrap-Pack-Request |
| 5.xx | Server Error | Transport TS 1.2.2 §6.7 (text after Table 6.7-4): if an operation cannot be completed and no 4.xx code describes the reason, a 5.xx code MUST be returned |

Transport TS 1.2.2 §6.7 states that the codes in Tables 6.7-1 to 6.7-4 "are the only valid response
codes defined in the LwM2M Enabler", plus the generic 5.xx rule above.

**Key CoAP Options for LwM2M:**
| Option | Number | Purpose in LwM2M |
|--------|--------|-------------------|
| Uri-Path | 11 | Resource addressing (/rd, /bs, /bspack, /dp, object paths); one option per path segment |
| Content-Format | 12 | Payload format — see numeric codes in [Content-Format Negotiation](#content-format-negotiation) |
| Accept | 17 | Server's preferred response format (Read/Observe); 40 (link-format) selects Discover |
| Uri-Query | 15 | Register: `ep`, `lt`, `lwm2m`, `b`, `Q`, `sms`, `pid`; Update: `lt`, `b`, `Q`, `sms`, `pid`; Bootstrap-Request: `ep`, `pct`; Bootstrap-Pack-Request: `ep`, `acc`; Discover: `depth`; Write-Attributes/Observe: `pmin`, `pmax`, `gt`, `lt`, `st`, `epmin`, `epmax`, `edge`, `con`, `hqmax` |
| Observe | 6 | 0 = register observation, 1 = cancel observation (Transport TS 1.2.2 Table 6.4.5-1) |
| Block1 | 27 | Blockwise request body (firmware push via Write to /5/0/0) |
| Block2 | 23 | Blockwise response body |
| Size1 | 60 | Request body size hint |
| Size2 | 28 | Response body size hint |
| ETag | 4 | Cache validation |
| Location-Path | 8 | Registration location (`rd`,`{id}`) and Create result (`{O}`,`{I}`); one option per segment, reassemble in order |

---

## LwM2M Interface to CoAP Mapping

### Registration Interface (Transport TS 1.2.2 §6.4.3, Tables 6.4.3-1 and 6.4.3-2)
| Operation | CoAP Method | URI | Key Options |
|-----------|------------|-----|-------------|
| Register | POST | /rd | Uri-Query: `ep` (optional, v1.1+), `lt` (mandatory), `lwm2m` (mandatory), `b`, `Q`, `sms`, `pid` (v1.2); payload = Object/Instance list in CoRE Link Format |
| Update | POST | /rd/{id} | Uri-Query: `lt`, `b`, `Q`, `sms`, `pid` — only parameters that changed (Core TS 1.2.2 §6.2.2); optional payload with changed Object list |
| De-register | DELETE | /rd/{id} | — |

### Device Management & Service Enablement Interface (Transport TS 1.2.2 Table 6.4.4-1)
| Operation | CoAP Method | URI | Content-Format |
|-----------|------------|-----|----------------|
| Read | GET | /{O}, /{O}/{I}, /{O}/{I}/{R}, /{O}/{I}/{R}/{RI} | Accept (optional) selects response format; 4.06 if unsupported |
| Discover | GET | /{O}, /{O}/{I}, /{O}/{I}/{R} + optional `?depth=0..3` | Accept: application/link-format (40) |
| Write (Replace) | PUT | /{O}/{I}, /{O}/{I}/{R}, /{O}/{I}/{R}/{RI} | Resource level: Plain Text, Opaque, CBOR, TLV, LwM2M CBOR, SenML CBOR/JSON; Instance level: TLV, LwM2M CBOR, SenML CBOR/JSON |
| Write (Partial Update) | POST | /{O}/{I}; /{O}/{I}/{R} only for Multiple-Instance Resources | TLV, LwM2M CBOR, SenML CBOR/JSON |
| Execute | POST | /{O}/{I}/{R} | none, or text/plain arguments (ABNF in Core TS 1.2.2 §6.3.5) |
| Create | POST | /{O} | TLV, LwM2M CBOR, SenML CBOR/JSON |
| Delete | DELETE | /{O}/{I}, /{O}/{I}/{R}/{RI} | — |
| Write-Attributes | PUT | /{O}, /{O}/{I}, /{O}/{I}/{R}, /{O}/{I}/{R}/{RI} | Uri-Query: pmin, pmax, gt, lt, st, epmin, epmax, edge, con, hqmax; no payload |
| Read-Composite (v1.1+) | FETCH | / | Request: SenML CBOR/JSON or SenML-ETCH CBOR/JSON; Accept: LwM2M CBOR, SenML CBOR/JSON |
| Write-Composite (v1.1+) | iPATCH | / | LwM2M CBOR, SenML CBOR/JSON, SenML-ETCH CBOR/JSON |

Transport TS 1.2.2 §6.4.4: a POST on a bare Object path is always Create — Write at Object level
cannot be distinguished and is therefore not supported on the CoAP binding.

### Information Reporting Interface (Transport TS 1.2.2 Table 6.4.5-1)
| Operation | CoAP Method | URI |
|-----------|------------|-----|
| Observe | GET + Observe:0 | /{O}, /{O}/{I}, /{O}/{I}/{R}, /{O}/{I}/{R}/{RI} (+ optional attribute query) |
| Cancel Observation | GET + Observe:1 (same path), or RST to a notification | same path as the Observe |
| Notify | asynchronous 2.05 response (CON or NON) | — |
| Observe-Composite (v1.1+) | FETCH + Observe:0 | / (+ optional pmin/pmax/epmin/epmax/con query) |
| Cancel Observation-Composite (v1.1+) | FETCH + Observe:1, payload = exactly the same paths as the Observe-Composite | / |
| Send (v1.1+) | POST | /dp — Content-Format MUST be LwM2M CBOR, SenML CBOR or SenML JSON |

---

## Registration Flow

```
Client                                    Server
  │                                          │
  │── POST /rd?ep=node&lt=300             ──►│
  │   &lwm2m=1.2&b=U&Q                       │
  │   Content-Format: application/link-format│
  │   Payload: </1/0>,</3/0>,</4/0>,</5>     │
  │                                          │
  │◄── 2.01 Created ────────────────────────│
  │    Location-Path: rd                     │
  │    Location-Path: a1b2c3   (→ /rd/a1b2c3)│
  │                                          │
  │    ... time passes (before lt expires)   │
  │                                          │
  │── POST /rd/a1b2c3 ─────────────────────►│
  │   (Registration Update — no params       │
  │    unless something changed)             │
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

**Registration parameters (Core TS 1.2.2 Table 6.2.1-1):**
- `ep` — Endpoint Client Name. Optional since v1.1 (the Register "SHOULD include" it); it MUST be
  supplied when the security protocol does not provide an authenticated identifier (Core TS 1.2.2
  §7.4). Mandatory in v1.0 (TS 1.0.2 §5.3.1).
- `lt` — Lifetime in seconds. **Mandatory**; MUST equal the value of /1/x/1 of the matching Server
  Object Instance. There is no default.
- `lwm2m` — Enabler version. **Mandatory**; MUST be the approved version number (e.g. `1.2`). The
  server MUST refuse the registration if it does not support that version (Core TS 1.2.2 §6.2.1) —
  4.12 Precondition Failed (Transport TS 1.2.2 Table 6.7-2).
- `b` — Binding. Optional, default `U`. Values are combinations of `U` (UDP), `M` (MQTT), `H` (HTTP),
  `T` (TCP), `S` (SMS), `N` (Non-IP) per Core TS 1.2.2 §6.2.1.2 and SHOULD match /3/0/16. The v1.0
  values `UQ`/`SQ`/`UQS` are superseded: since v1.1 Queue Mode is the separate `Q` flag.
- `Q` — Queue Mode flag (v1.1+), present without a value.
- `sms` — MSISDN (or External Identifier) for the SMS binding or SMS registration-update trigger.
- `pid` — Profile ID (v1.2): object list by reference; ASCII per the ABNF in Core TS 1.2.2 §6.2.1
  (`oma:`, `v:` or `<suite>:<hex>`); may be repeated and mixed with a payload list. If the server
  cannot resolve it, it replies with an error (4.09 in Transport Table 6.7-2) and the client MUST
  re-register with the full list.

The server MUST support every parameter except Profile ID; the client MUST implement `lwm2m`, `lt`
and the Object/Instance list and MAY implement the rest (Core TS 1.2.2 §6.2.1).

**Registration payload** is CoRE Link Format (application/link-format, RFC 6690) listing the Objects
supported and Object Instances available. Object versions are carried via the `ver` link parameter
when required by Core TS 1.2.2 §7.2. **Security Object /0, OSCORE Object /21 and COSE Object /23
MUST NOT appear in the list** (Core TS 1.2.2 Table 6.2.1-1). Supported optional content formats MAY
be advertised on the root link, e.g. `</>;ct="110 112 60"`.

---

## Bootstrap Flows

### Bootstrap Sequence (Core TS 1.2.2 §6.1.4)
The client MUST follow this order:
1. If a Smartcard is present, try Bootstrap from Smartcard. Server Initiated Bootstrap attempts MUST
   be ignored until Smartcard or Factory Bootstrap has been tried.
2. Otherwise try Factory Bootstrap (same rule on ignoring Server Initiated Bootstrap).
3. If any Server Object Instances exist, register with the configured LwM2M Server(s).
4. If registration fails to all servers, or no Server Object Instance exists, perform Client
   Initiated Bootstrap.
5. Server Initiated Bootstrap remains possible only while the client retains the Bootstrap-Server
   Account.

### Client-Initiated Bootstrap (Transport TS 1.2.2 §6.4.2, Figure 6.4.2-1)
```
Client                              Bootstrap-Server
  │                                       │
  │── POST /bs?ep=myDevice[&pct=112] ───►│   pct = preferred Content Format (v1.1+)
  │                                       │
  │◄── 2.04 Changed ───────────────────  │
  │                                       │
  │◄── DELETE / ────────────────────────  │  (clean slate — spares the Bootstrap-Server
  │── 2.02 Deleted ────────────────────► │   Account and the Device Object /3/0)
  │                                       │
  │◄── PUT /0/1 (Security for Server) ─  │  (Bootstrap-Write: any format from §7.5,
  │── 2.04 Changed ────────────────────► │   written regardless of existence/ACL)
  │                                       │
  │◄── PUT /1/0 (Server Object) ───────  │
  │── 2.04 Changed ────────────────────► │
  │                                       │
  │◄── PUT /2/0 (ACL, if multi-server) ─ │
  │── 2.04 Changed ────────────────────► │
  │                                       │
  │◄── POST /bs (Bootstrap-Finish) ────  │
  │── 2.04 Changed ────────────────────► │  (4.06 Not Acceptable if the loaded
  │                                       │   configuration is inconsistent, Core §6.1.6)
  │   [Client now registers with Server]  │
```

The Bootstrap-Server MUST send Bootstrap-Finish after the last Bootstrap-Write (Transport TS 1.2.2
§6.4.2). The client treats the procedure as failed if no Bootstrap-Finish arrives within
EXCHANGE_LIFETIME (Core TS 1.2.2 §6.1.6). Bootstrap-Discover and Bootstrap-Read are also available;
Bootstrap-Read is limited to Object 1 and Object 2.

### Bootstrap-Pack-Request (v1.2; `acc` parameter v1.2.1)
Transport TS 1.2.2 Table 6.4.2-1 / Figure 6.4.2-2; Core TS 1.2.2 §6.1.7.7.
```
Client                              Bootstrap-Server
  │                                       │
  │── GET /bspack?ep=myDevice ─────────►│   optional &acc={Bootstrap-Server Account
  │   Accept: 112 (SenML CBOR) | 110 ... │   Object Instances}; Accept MUST be SenML
  │                                       │   CBOR (112), SenML JSON (110) or
  │◄── 2.05 Content ──────────────────  │   LwM2M CBOR (11544)
  │    Payload: complete bootstrap config │
  │    (Security + Server [+ ACL] objects │
  │     in one SenML or LwM2M-CBOR pack)  │
```

- **No Bootstrap-Finish** follows a Bootstrap-Pack (Transport TS 1.2.2 §6.4.2).
- The pack replaces Objects whose IDs it carries and leaves others untouched; the Bootstrap-Server
  Account (and its OSCORE instance) and /3/0 are never deleted (Core TS 1.2.2 §6.1.7.7).
- **Fallback:** a Bootstrap-Server that does not implement the operation MUST answer "not
  implemented" (5.01); one that implements it but does not want the client to use a pack MUST answer
  "method not allowed" (4.05). After *any* error response the client MUST continue with
  Bootstrap-Request (Core TS 1.2.2 §6.1.7.7; codes in Transport TS 1.2.2 Table 6.7-1). If the pack
  is inconsistent the client MUST retry with Bootstrap-Request (Core §6.1.6). With no response within
  EXCHANGE_LIFETIME the client MAY retry the Bootstrap-Pack-Request.

Reduces the bootstrap exchange from many round-trips to a single request-response.

---

## Observation & Notification

### Observation Attributes (Core TS 1.2.2 Table 7.3.2-1)
| Attribute | Meaning | Attachment levels | Since |
|-----------|---------|-------------------|-------|
| pmin | Minimum period between notifications (s); default = /1/x/2 or 0 | Object / Instance / Resource / Resource Instance | v1.0 |
| pmax | Maximum period between notifications (s); 0 = ignored | Object / Instance / Resource / Resource Instance | v1.0 |
| gt | Greater Than — notify when the value crosses this threshold | Resource / Resource Instance (numeric) | v1.0 |
| lt | Less Than — notify when the value crosses this threshold | Resource / Resource Instance (numeric) | v1.0 |
| st | Step — notify when the change since the last notification ≥ st | Resource / Resource Instance (numeric) | v1.0 |
| epmin | Minimum evaluation period (s) | Object / Instance / Resource / Resource Instance | v1.1 |
| epmax | Maximum evaluation period (s) | Object / Instance / Resource / Resource Instance | v1.1 |
| edge | `edge=0` falling edge, `edge=1` rising edge of a Boolean resource | Resource / Resource Instance (Boolean) | v1.2 |
| con | `con=1` notifications MUST use confirmable transport; a multi-component notification is confirmable if any component has con=1 | Object / Instance / Resource / Resource Instance | v1.2 |
| hqmax | Maximum Historical Queue — entries stored while offline/disabled; oldest dropped when full; only effective when /1/x/6 (Notification Storing) is true | Object / Instance / Resource / Resource Instance | v1.2 |

Inheritance and precedence (Core TS 1.2.2 §7.3.2): an attribute applies to the level it is attached
to and all lower levels; a lower-level attachment overrides an upper one. Server MUST and client
SHOULD support all of them.

**Invariants (Core TS 1.2.2 §7.3.2 and §6.4.2):**
- `lt < gt` and `lt + 2·st < gt`; a Write-Attributes, Observe or Observe-Composite that violates
  either MUST be rejected (4.00 Bad Request per Transport TS 1.2.2 §6.4.4).
- If both are set, `epmin < epmax`.
- `pmax` smaller than `pmin` (or `pmax = 0`) is ignored for that observation.

### Notification Conditions (Core TS 1.2.2 §6.4.2, Notify Operation)
A Notify MUST be sent when the observed item changes and all conditions (pmin, pmax, gt, lt, st)
are met, **or** when the pmax timer expires. In practice:
1. pmin has expired since the last notification (a change inside pmin is sent as soon as pmin expires)
2. at least one Change Value Condition is satisfied (gt/lt crossing, st step, edge transition)
3. OR pmax expires with no change

The response to the Observe itself is the initial notification; each Notify restarts pmin/pmax.
Attributes attached below the observed level are ignored for that observation.

---

## Composite Operations (v1.1+)

Introduced in LwM2M 1.1 (Core TS 1.1 §6.3.8 Read-Composite, §6.3.9 Write-Composite, §6.4.4
Observe-Composite, §6.4.5 Cancel Observation-Composite); same numbering in Core TS 1.2.2. Client
support is optional (MAY); if Observe-Composite is supported, Cancel Observation-Composite MUST be.

### Read-Composite (Core TS 1.2.2 §6.3.8)
CoAP FETCH (RFC 8132) on `/`. The request body is a SenML Pack with Base Name / Name fields only
(RFC 8790), in SenML CBOR, SenML JSON, SenML-ETCH CBOR or SenML-ETCH JSON; the response format is
chosen via Accept from LwM2M CBOR, SenML CBOR or SenML JSON (Transport TS 1.2.2 Table 6.4.4-1).

**Best effort, non-atomic:** paths that have no valid value or are not readable are simply omitted
from the response; there is no per-path error marker.

```
Client                          Server
  │                               │
  │◄── FETCH / ─────────────────  │
  │    Content-Format: SenML-CBOR │
  │    Payload: [{n:"/3/0/0"},    │
  │              {n:"/3/0/9"},    │
  │              {n:"/4/0/2"}]    │
  │                               │
  │── 2.05 Content ────────────► │
  │   Content-Format: SenML-CBOR  │
  │   Payload: [{bn:"/3/0/",      │
  │              n:"0", vs:"Mfr"},│
  │             {n:"9", v:85},    │
  │             {bn:"/4/0/",      │
  │              n:"2", v:-72}]   │
```

### Write-Composite (Core TS 1.2.2 §6.3.9)
CoAP iPATCH (RFC 8132) on `/`; body in LwM2M CBOR, SenML CBOR/JSON or SenML-ETCH CBOR/JSON carrying
path-value pairs across any Instances of any Objects. Resources not listed are untouched.
**Atomic:** the client MUST reject the whole request if it cannot write every value (existence,
write access and resource type are checked first); partial success is not allowed.

### Observe-Composite / Cancel Observation-Composite (Core TS 1.2.2 §6.4.4, §6.4.5)
FETCH + Observe:0 on `/` with the path list in the body; optional pmin/pmax/epmin/epmax/con as
Object-level observation attributes in the query. Every notification carries the values of **all**
listed paths, even those that did not trigger. Cancel = FETCH + Observe:1 with exactly the same path
list (Transport TS 1.2.2 Table 6.4.5-1).

---

## Send Operation (v1.1+)

Introduced in LwM2M 1.1 (Core TS 1.1 §6.3.10, then part of the DM&SE interface); moved to the
Information Reporting interface as Core TS 1.2 §6.4.6 (same number in 1.2.2). Client and server
MAY support it.

Device-initiated data push: the client sends data to the server without a prior request.

```
Client                          Server
  │                               │
  │── POST /dp ────────────────► │   (CON, or NON — Transport §6.8.1)
  │   Content-Format: SenML-CBOR  │
  │   Payload: [{bn:"/3303/0/",   │
  │              n:"5700",v:23.5}, │
  │             {bn:"/3304/0/",   │
  │              n:"5700",v:65}]  │
  │                               │
  │◄── 2.04 Changed ────────────  │
```

- Content-Format MUST be LwM2M CBOR, SenML CBOR or SenML JSON (Transport TS 1.2.2 §6.4.5).
- Only Object Instances registered with that server, and only Resources the server may Read, may be
  reported (Core TS 1.2.2 §6.4.6).
- Response codes (Transport TS 1.2.2 Table 6.7-4): 2.04 Changed; 4.00 Bad Request; 4.04 Not Found
  when a reported Object was not registered to that server.
- **Mute Send** /1/x/23 (Boolean, RW, Optional; OMA-SUP-XML_1 v1.2): when **true or absent** the
  client's Send capability is **de-activated**; `false` activates it.

---

## Queue Mode

Queue Mode is essential for sleepy devices (NB-IoT PSM, LTE-M eDRX). Transport TS 1.2.2 §6.5: the
server MUST and the client SHOULD support it.

**Signalling:** v1.1+ clients add the `Q` Uri-Query flag to Register/Update alongside the transport
binding (`b=U&Q`); v1.0 encoded it inside the binding (`b=UQ`, `SQ`, `UQS` — TS 1.0.2 §5.3.1).

**Behaviour:**
1. Client registers with `b=U&Q` (UDP + Queue Mode)
2. Client enters sleep (radio off / PSM)
3. Server queues any pending operations (Read, Write, Execute, Observe setup) instead of sending them
4. Client wakes and sends a Confirmable Registration Update
5. Server treats the Update as the "awake" signal and flushes the queued operations
6. Client processes them; it is RECOMMENDED to stay reachable at least MAX_TRANSMIT_WAIT after its
   last CoAP message before sleeping again (Transport TS 1.2.2 §6.5)

**With DTLS CID (RFC 9146; Transport TS 1.2.2 §5.2.8, v1.2):**
After PSM sleep and NAT rebinding, the client's source IP/port changes. Without CID, this breaks the
DTLS association and forces a full re-handshake. With CID, the server matches the CID in the DTLS
record header to the existing session — no re-handshake needed.

**NAT/firewall window:** Transport TS 1.2.2 §6.2 requires a firewall to allow return UDP/DTLS packets
for at least 240 seconds; §6.3 notes TCP bindings typically survive longer in NATs.

---

## Blockwise Transfer

For payloads exceeding the CoAP message size (typically ~1024 bytes for UDP), RFC 7959 blockwise
transfer is used. Transport TS 1.2.2 §6.1: blockwise MUST be supported by the server, and by the
client when it implements the Firmware Update Object (ID 5); the client MAY use it for other
objects. The Request-Tag option SHOULD be used to keep concurrent blockwise requests apart (§5.5).

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
Server pushes firmware to client via Write to /5/0/0 (Package resource) using Block1; intermediate
blocks are acknowledged with 2.31 Continue (Transport TS 1.2.2 Table 6.7-3).

---

## Content-Format Negotiation

**Numeric Content-Formats (Core TS 1.2.2 Tables 7.5-1 to 7.5-3):**
| Format | Media type | Code | Since |
|--------|-----------|------|-------|
| Plain Text | text/plain | 0 | 1.0 |
| CoRE Link Format | application/link-format | 40 | 1.0 |
| Opaque | application/octet-stream | 42 | 1.0 |
| TLV | application/vnd.oma.lwm2m+tlv | 11542 | 1.0 (legacy from 1.1) |
| LwM2M 1.0 JSON | application/vnd.oma.lwm2m+json | 11543 | 1.0 (legacy from 1.1) |
| CBOR (single value) | application/cbor | 60 | 1.1.1 (ERELD 1.1.1 §3; Core TS 1.1 Table 7.4-2 lists only 110/112, Core TS 1.2.2 Table 7.5-2 groups it under 1.1) |
| SenML JSON | application/senml+json | 110 | 1.1 |
| SenML CBOR | application/senml+cbor | 112 | 1.1 |
| LwM2M CBOR | application/vnd.oma.lwm2m+cbor | 11544 | 1.2 |
| SenML-ETCH JSON | application/senml-etch+json | 320 | 1.2 |
| SenML-ETCH CBOR | application/senml-etch+cbor | 322 | 1.2 |

**Requirements (Core TS 1.2.2 §7.5):**
- The server MUST support all data formats, including TLV.
- The client MUST support Plain Text, Opaque and CoRE Link Format, and SHOULD support at least one
  of SenML CBOR, SenML JSON or LwM2M CBOR. Clients at 1.1 or newer SHOULD NOT use the legacy TLV or
  1.0 JSON formats.
- Every payload MUST declare its format with Content-Format.
- The server MAY state a preference with Accept; if the client does not accept it the request is
  rejected (4.06 Not Acceptable, Transport TS 1.2.2 §6.4.4). Without Accept, the client uses its own
  preferred format.
- The client MAY advertise supported optional formats on the root link at registration:
  `</>;ct="110 112 60"` (Core TS 1.2.2 §6.2.1).

**Negotiation rules in practice:**
1. If Accept is present, the server is requesting that format
2. If the client cannot produce it, 4.06 Not Acceptable
3. Default when no Accept: TLV for v1.0 clients; the client's own preferred format for v1.1+
4. CBOR (60), SenML JSON and SenML CBOR require v1.1+
5. LwM2M CBOR and SenML-ETCH require v1.2+
6. Composite operations and Send require SenML (or SenML-ETCH for composite requests) or LwM2M CBOR

---

## Transport Binding Details

Transport TS 1.2.2 §6.8: the server MUST and the client SHOULD support the UDP binding. §6.1: CoAP
over TCP/TLS SHOULD be used for firewall/NAT traversal; CoAP over WebSockets MAY be used.

### CoAP/UDP (Binding "U") — Transport TS 1.2.2 §6.8.1
- Default transport, supported since v1.0; `coap://` port 5683 (NoSec), `coaps://` port 5684 (DTLS)
- Security: DTLS 1.2 (RFC 6347); DTLS 1.3 (RFC 9147) optional since v1.2
- DTLS CID (RFC 9146) for NAT traversal (v1.2, §5.2.8)
- All operations CON except Notify, Execute and Send, which MAY be NON
- Block size typically 1024 bytes (configurable)

### CoAP/TCP (Binding "T") — Transport TS 1.2.2 §6.8.2
- Added in v1.1 (RFC 8323); `coap+tcp://` port 5683, `coaps+tcp://` port 5684 (TLS 1.2/1.3)
- No message layer (CON/NON/ACK/RST) — delivery is reliable, no CoAP retransmission
- Block1/Block2 are still used for large payloads; RFC 8323 adds BERT (Block-wise Extension for
  Reliable Transport — block sizes above 1024 bytes) once both peers signal it in the CSM. "TCP needs
  no blockwise" is wrong — Transport §6.1 blockwise requirements apply to every CoAP binding
- Better for always-connected devices behind NATs/firewalls (§6.2, §6.3)
- Ping/Pong signalling for keepalive

### CoAP/WebSockets — Transport TS 1.2.2 §6.8.6 (v1.2.1)
- RFC 8323 WebSockets binding; `coap+ws://` port 80 (NoSec), `coaps+ws://` port 443 (TLS)
- MAY be used for firewall/NAT traversal (§6.1); no dedicated binding letter in the `b` parameter

### CoAP/SMS (Binding "S") — Transport TS 1.2.2 §6.8.3, §5.3
- Supported since v1.0; CoAP message placed in the SMS payload using 8-bit encoding
- 140-byte TPDU; SMS concatenation MAY be used for larger messages
- **CoAP retransmission is disabled** for this binding
- Client signals the binding with the `sms` registration parameter (MSISDN per 3GPP 23.003, no "+")
- **SMS Secured Packet mode MUST be supported when the SMS binding is used** (§5.3.2; 3GPP 31.115 /
  ETSI 102 225 Command Packets, AES SHOULD, single DES MUST NOT); NoSec is allowed only for
  debugging or for SMS triggering when the main channel is secured (§5.3, §5.3.1)
- Effective LwM2M payload per SMS: ~107 bytes without a Token, ~99 bytes with a Token
  (§5.3.2.1.1, DTLS 29-byte overhead model)
- An SMS from an MSISDN not recorded in the server account MUST be silently ignored (§5.3)
- Useful for SMS-triggered wake-up (Registration Update Trigger §6.6.1, Bootstrap Trigger §6.6.2)
  with main data over UDP

### CoAP over Non-IP (Binding "N") — Core TS 1.2.2 §6.2.1.2
- Added in v1.1; covers 3GPP CIoT non-IP delivery (NB-IoT / LTE-M, Transport TS 1.2.2 §6.8.5 and
  Appendix D) and LoRaWAN
- Typically very constrained payload sizes (50-240 bytes)
- OSCORE is often preferred over DTLS for Non-IP (avoids DTLS handshake overhead)

#### LoRaWAN — Transport TS 1.2.2 §6.8.4 and Appendix C (Normative), v1.1+
- The LoRaWAN end-device MUST be the LwM2M Client; LwM2M Server and Bootstrap-Server MUST be LoRaWAN
  Application Servers
- **NoSec mode**; security comes from the LoRaWAN link layer
- Server URI MUST be `lorawan://{FPort}` with FPort 1..255
- Register parameters MAY be omitted (Table 6.8.4-1): `ep` defaults to the DevEUI, `lt` defaults to
  2 592 000 s (30 days), `lwm2m` may be omitted when 1.1, binding is inherently Queue Mode, and the
  object list MAY be omitted on the first attempt — if the server cannot resolve it out-of-band it
  MUST reply 4.09 Conflict and the client SHOULD retry with the list
- Piggybacked responses MUST be used; CoAP retransmission is replaced by Appendix C logic:
  MAX_RETRANSMIT = 4, ACK_TIMEOUT = 300 s (App C.2), with RX-window re-opening via CoAP Empty
  messages (App C.3)

### MQTT (Binding "M") — Transport TS 1.2.2 §8 (v1.2)
- Works with MQTT 3.1.1 and MQTT 5 (no v5-only features used)
- Topic ABNF (§8.2): `topic = [ PREFIX "/" ] "lwm2m/" ( "bs" / "rd" ) "/" ENDPOINT` — `bs` for the
  Bootstrap interface, `rd` for all others
- Payloads are CBOR maps with integer keys (§8.7 Table 8.7-1: 1 operation, 2 token, 3 ep, 4 pct,
  5 uri, 6 paths, 7 payload, 8 lifetime, 9 version, 10 b, 11 sms, 12 pmin, 13 pmax, 14 gt, 15 st,
  16 epmin, 17 epmax, 18 result, 19 ct, 20 edge, 21 hqmax, 22 depth)
- Server URI in /0/x/0 MUST use `mqtt://` or `mqtts://` (§8.8); TLS protects the MQTT connection
  (§8.6)
- Configuration: Security Object R26 "MQTT Server" (Objlnk) → MQTT Server Object (ID 24); R27
  "LwM2M COSE Security" → COSE Object (ID 23). If the Security instance links to a COSE instance,
  the client MUST protect MQTT messages with COSE (§8.8)
- Useful for cloud platform integration; broker ACL and tenant isolation are recommended (§8.1)

### HTTP (Binding "H") — Transport TS 1.2.2 §7 (v1.2)
- Optional to implement and to use; SHOULD use HTTP/1.1, MAY use HTTP/2; secured with TLS
- Information Reporting interface is limited to **Send** (`POST /dp`, §7.1.5) — no Observe/Notify
- Queue Mode as for CoAP (§7.2); HTTP status codes in §7.3
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

# Bootstrap-Pack-Request: GET to /bspack
tshark -r capture.pcap -Y 'coap.code == 1 && coap.opt.uri_path == "bspack"'

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
