# Operation-to-CoAP mapping

This reference explains *how* the LwM2M interface operations map onto CoAP methods,
options, and URI levels — and where that mapping is overloaded, subtle, or a frequent
source of interoperability bugs. The companion file `protocol-details.md` carries the
interface→method *table*; this file fills in the *semantics* behind each row, the
wire-format details, and the exact conformance response codes.

Section numbers refer to OMA-TS-LightweightM2M_Core-V1_2_2-20240613-A ("Core TS 1.2.2") and
OMA-TS-LightweightM2M_Transport-V1_2_2-20240613-A ("Transport TS 1.2.2") unless stated otherwise.
Operation semantics live in Core TS 1.2.2 §6.3.x / §6.4.x; the CoAP method/URI/format mapping lives
in Transport TS 1.2.2 §6.4.x; response codes in Transport TS 1.2.2 §6.7.

The recurring theme is that several LwM2M operations share a single CoAP method and are
distinguished only by **target-path depth**, **CoAP options**, **request body**, or the
**target Resource's definition** — not by the method verb alone. A correct dispatcher therefore
cannot branch on the CoAP method in isolation.

## Table of contents

- [Master mapping table](#master-mapping-table)
- [A. POST is overloaded — disambiguate by target-path depth and resource definition](#a-post-is-overloaded--disambiguate-by-target-path-depth-and-resource-definition)
- [B. PUT vs POST write semantics, and a correctness trap](#b-put-vs-post-write-semantics-and-a-correctness-trap)
- [C. Write scope and content-format admissibility by URI depth](#c-write-scope-and-content-format-admissibility-by-uri-depth)
- [D. GET is overloaded — disambiguate by CoAP options](#d-get-is-overloaded--disambiguate-by-coap-options)
- [E. Composite operations are body-addressed](#e-composite-operations-are-body-addressed)
- [F. Create wire-format — the URI / payload / Location-Path triad](#f-create-wire-format--the-uri--payload--location-path-triad)
- [G. Registration interface contract](#g-registration-interface-contract)
- [H. Write-Attributes](#h-write-attributes)
- [I. Send](#i-send)
- [J. Minimal-client conformance — always answer, even to refuse](#j-minimal-client-conformance--always-answer-even-to-refuse)
- [K. Conformance response-code quick table](#k-conformance-response-code-quick-table)
- [Key specification references](#key-specification-references)
- [Related files](#related-files)

## Master mapping table

The single most important column below is **URI level**: it is what disambiguates the
overloaded methods. Read it together with [§A](#a-post-is-overloaded--disambiguate-by-target-path-depth-and-resource-definition)
and [§D](#d-get-is-overloaded--disambiguate-by-coap-options). Source: Transport TS 1.2.2
Tables 6.4.3-2, 6.4.4-1 and 6.4.5-1; success codes from Tables 6.7-2 to 6.7-4.

| Operation | CoAP method | Disambiguator | URI level | Success code | Notes |
|-----------|-------------|---------------|-----------|--------------|-------|
| Read | GET | no `Accept:40`, no Observe | `/o`, `/o/i`, `/o/i/r`, `/o/i/r/ri` | 2.05 Content | Returns values in the negotiated format |
| Discover | GET | `Accept: application/link-format` (CT 40) | `/o`, `/o/i`, `/o/i/r` (+ `?depth=0..3`) | 2.05 Content | Returns CoRE Links + attributes, not values |
| Observe (register) | GET | Observe option = 0 | `/o`, `/o/i`, `/o/i/r`, `/o/i/r/ri` | 2.05 Content | Notifications follow; optional attribute query |
| Observe (cancel, active) | GET | Observe option = 1 | same as registration | 2.05 Content | Active cancel; RST to a notification also cancels |
| Write (Replace) | PUT | — | `/o/i`, `/o/i/r`, `/o/i/r/ri` | 2.04 Changed | Omitted Resources / Resource Instances replaced |
| Write (Partial Update) | POST | body in TLV / LwM2M CBOR / SenML | `/o/i`; `/o/i/r` **only for Multiple-Instance Resources** | 2.04 Changed | Omitted Resources untouched |
| Write-Attributes | PUT | Uri-Query params, no payload | `/o`, `/o/i`, `/o/i/r`, `/o/i/r/ri` | 2.04 Changed | pmin/pmax/gt/lt/st/epmin/epmax/edge/con/hqmax |
| Execute | POST | target Resource is Executable (single-instance) | `/o/i/r` | 2.04 Changed | Optional text/plain args |
| Create | POST | bare Object URI | `/o` | 2.01 Created | Location-Path → new instance |
| Delete | DELETE | — | `/o/i`, `/o/i/r/ri` | 2.02 Deleted | Resource Instance delete is v1.1+ |
| Read-Composite (v1.1+) | FETCH | body-addressed | `/` (root) | 2.05 Content | Paths in SenML / SenML-ETCH body; best effort |
| Write-Composite (v1.1+) | iPATCH | body-addressed | `/` (root) | 2.04 Changed | Paths + values in body; atomic |
| Observe-Composite (v1.1+) | FETCH | Observe option = 0, body-addressed | `/` | 2.05 Content | Notifications carry all listed paths |
| Cancel Observation-Composite (v1.1+) | FETCH | Observe option = 1, same path list | `/` | 2.05 Content | |
| Register | POST | — | `/rd` + Uri-Query | 2.01 Created | Location-Path `rd`,`{id}` |
| Update | POST | — | `/rd/{id}` (+ changed Uri-Query) | 2.04 Changed | |
| De-register | DELETE | — | `/rd/{id}` | 2.02 Deleted | |
| Send (v1.1+) | POST | — | `/dp` | 2.04 Changed | Client-initiated; SenML or LwM2M CBOR body |

`o` = Object ID, `i` = Object Instance ID, `r` = Resource ID, `ri` = Resource Instance ID throughout.
Resource-Instance-level addressing (`/o/i/r/ri`) exists from v1.1 (Core TS 1.1 §6.3.1).

## A. POST is overloaded — disambiguate by target-path depth and resource definition

POST is the most overloaded method in LwM2M. The same verb means three different
operations depending on the depth of the target URI — and at Resource depth, on the
**definition of the target Resource**. This is the single most common LwM2M↔CoAP dispatch bug.

```
POST /{ObjId}                  -> Create                     (bare Object URI)
POST /{ObjId}/{InstId}         -> Write (Partial Update)     (Object Instance URI, TLV/LwM2M CBOR/SenML body)
POST /{ObjId}/{InstId}/{ResId} -> Execute                    if the Resource is Executable
                               -> Write (Partial Update)     if the Resource is a Multiple-Instance Resource
```

- **POST on a bare Object URI** `/{ObjId}` = **Create** (Core TS 1.2.2 §6.3.6; Transport TS
  1.2.2 Table 6.4.4-1). Transport §6.4.4 notes that Create at Object level cannot be distinguished
  from a Write at Object level, so Write at Object level is not supported on the CoAP binding.
- **POST on an Object Instance URI** `/{ObjId}/{InstId}` with a TLV, LwM2M CBOR or SenML body =
  **Write (Partial Update)** (Core TS 1.2.2 §6.3.3): only the Resources carried in the body are
  written; **omitted Resources are left unchanged.**
- **POST on a Resource URI** `/{ObjId}/{InstId}/{ResId}`:
  - if the Resource is **Executable** = **Execute** (Core TS 1.2.2 §6.3.5), with optional
    plain-text arguments in the body (ABNF `arglist = arg *( "," arg )`, each DIGIT at most once);
  - if the Resource is a **Multiple-Instance Resource** = **Write (Partial Update)** of the Resource
    Instance array (Transport TS 1.2.2 Table 6.4.4-1: "POST … /{Resource ID} only for
    Multiple-Instance Resources"); Instances in the body are created or overwritten, none deleted
    (Core TS 1.2.2 §6.3.3).
  Execute is only valid on individual Resources; Execute aimed at an Object Instance or a Resource
  Instance MUST be answered with an error (Core TS 1.2.2 §6.3.5).

A dispatcher must therefore branch a POST **by URI depth first, then by the Resource's operation
flags from the Object definition** before doing anything else. A real, fixed bug: routing *every*
instance-scoped POST to Create makes a client wrongly return `4.05 Method Not Allowed` to
legitimate Partial-Update writes, because Create is not valid against an already-existing
Object Instance URI.

## B. PUT vs POST write semantics, and a correctness trap

PUT and POST both perform a Write against an Object Instance, but with **opposite
semantics for omitted Resources** (Core TS 1.2.2 §6.3.3; Transport TS 1.2.2 §6.4.4 "CoAP PUT is
used for Replace. CoAP POST or CoAP iPatch is used for Partial Update"):

- **PUT on an Object Instance** = **Write (Replace)**: Resources *not* present in the
  payload **must be reset** to their default/initial values (or removed if optional).
  The instance after the write reflects only what the payload described. On a
  Multiple-Instance Resource the whole Resource Instance array is replaced.
- **POST on an Object Instance** = **Write (Partial Update)**: Resources *not* present
  in the payload are **left untouched**. Deleting Resource Instances via Partial Update is not
  possible.

```
Before:  /3/0 = { 0:"Acme", 1:"M-1", 13:1700000000 }

PUT  /3/0  body={ 0:"Acme" }    -> Replace -> { 0:"Acme" }  (1,13 reset/removed)
POST /3/0  body={ 0:"Acme" }    -> Partial -> { 0:"Acme", 1:"M-1", 13:1700000000 }
```

**Common bug:** routing both PUT and POST into a single writer that only applies the
carried Resources makes Replace silently behave like Partial Update — omitted Resources
are wrongly preserved. Flag this whenever PUT and POST share a write code path: the
Replace branch must additionally reset/remove every Resource absent from the payload.

A Write MUST be rejected with "method not allowed" if **any** Resource in the payload does not
support Write (Core TS 1.2.2 §6.3.3), and with 4.00 if a value fails its type, range or Objlnk
check, or 4.15 if the format is unsupported (Transport TS 1.2.2 §6.4.4, Table 6.7-3).

## C. Write scope and content-format admissibility by URI depth

A Write must address **at least an Object Instance**. A Write targeting a bare Object
URI `/{ObjId}` is out of scope on the CoAP binding and is rejected `4.05 Method Not Allowed`
(Transport TS 1.2.2 §6.4.4: path rules from Core not respected → 4.05).

| Write target | URI | Method | Admissible content-formats (Transport TS 1.2.2 §6.4.4, Table 6.4.4-1) |
|--------------|-----|--------|-------------------------------------------------------------------|
| Resource Instance | `/o/i/r/ri` | PUT | Plain Text (0), Opaque (42), CBOR (60), TLV (11542), LwM2M CBOR (11544), SenML CBOR (112), SenML JSON (110) |
| Resource (single value) | `/o/i/r` | PUT | Plain Text (0), Opaque (42), CBOR (60), TLV, LwM2M CBOR, SenML CBOR, SenML JSON |
| Resource (Multiple-Instance array) | `/o/i/r` | PUT (replace) or POST (partial) | TLV, LwM2M CBOR, SenML CBOR, SenML JSON (Core TS 1.2.2 §6.3.3: structured format MUST be used when more than one value, or Resource Instances are allocated/deleted) |
| Instance-level | `/o/i` | PUT (replace) or POST (partial) | TLV, LwM2M CBOR, SenML CBOR, SenML JSON |
| Object-level | `/o` | — | not a valid Write target → 4.05 |

Plain Text, Opaque and CBOR (60) are admissible only for a single value because they carry **no
identifier**: a Plain Text value must be re-parsed into the Resource's *declared* data type from the
Object definition (Core TS 1.2.2 §7.5.1, Appendix C; see `objects.md`). TLV, LwM2M CBOR and SenML are
**self-describing** — each item carries its identifier and value encoding — so they are admissible
wherever multiple Resources or Resource Instances may appear.

**Default/mandatory formats (Core TS 1.2.2 §7.5):** the client MUST support Plain Text, Opaque and
CoRE Link Format and SHOULD support at least one of SenML CBOR, SenML JSON or LwM2M CBOR; clients at
1.1 or newer SHOULD NOT use the legacy TLV or 1.0 JSON formats. The server MUST support all formats,
TLV included. Every payload MUST carry a Content-Format option.

## D. GET is overloaded — disambiguate by CoAP options

GET maps to three operations, distinguished by CoAP **options**, never by the verb:

```
GET, Accept: application/link-format (CT 40)  -> Discover  (returns links)
GET, Observe option present                   -> Observe   (0=register, 1=cancel)
GET, neither of the above                     -> Read      (returns values)
```

- **GET with `Accept: application/link-format`** (CT 40) = **Discover** (Core TS 1.2.2 §6.3.2;
  Transport TS 1.2.2 Table 6.4.4-1): returns the Object/Instance/Resource/Resource-Instance link
  list in CoRE Link Format (RFC 6690) with attached or inherited attributes, **not** Resource
  values. The optional `?depth={0..3}` query (Table 6.3.2-1/6.3.2-2) bounds the listing relative to
  the target; defaults are 2 when only an Object is given, else 1. A client that does not support
  `depth` MUST ignore it and use the default.
- **GET carrying a CoAP Observe option** (RFC 7641 §3.1) = **Observe**: Observe option value
  **0 registers** an observation, **1 cancels** it (active cancellation) — Transport TS 1.2.2
  Table 6.4.5-1; semantics in Core TS 1.2.2 §6.4.1 and §6.4.3. A RST in reply to a notification also
  cancels (Transport Figure 6.4.5-1). Observation attributes may ride in the Uri-Query of the
  Observe; when present they override attached attributes for that observation (Core §6.4.1).
- **Plain GET** — no `Accept: 40` and no Observe option — = **Read** (Core TS 1.2.2 §6.3.1):
  returns values. An Accept with any other Content-Format selects the response encoding; 4.06 if
  the client cannot produce it (Transport TS 1.2.2 §6.4.4).

Note that Read, Discover and Observe can each apply at Object, Object Instance, Resource (and, for
Read/Observe, Resource Instance) level, so URI depth does **not** disambiguate GET — only the
options do.

## E. Composite operations are body-addressed

Composite operations **(v1.1+; Core TS 1.1 §6.3.8/§6.3.9/§6.4.4/§6.4.5, unchanged numbering in
1.2.2)** break the usual rule that the target lives in the URI. Client support is optional (MAY).

- **Read-Composite** = CoAP **FETCH**; **Write-Composite** = CoAP **iPATCH** (RFC 8132; Core TS
  1.2.2 §6.3.8 and §6.3.9; Transport TS 1.2.2 Table 6.4.4-1).
- Both address the **root** `/` and carry their **target resource paths inside the SenML request
  body**, not in `Uri-Path`.
- Request formats: Read-Composite — SenML CBOR, SenML JSON, SenML-ETCH CBOR (322) or SenML-ETCH
  JSON (320), with Accept choosing LwM2M CBOR, SenML CBOR or SenML JSON for the response;
  Write-Composite — LwM2M CBOR, SenML CBOR/JSON or SenML-ETCH CBOR/JSON.

```
FETCH /              (no Uri-Path)
Content-Format: application/senml+cbor
Body (SenML): [ {"n":"/3/0/0"}, {"n":"/3/0/1"}, {"n":"/6/0/0"} ]
```

A dispatcher must route **FETCH/iPATCH before parsing the URI**. If it parses the URI
first, a deliberately `Uri-Path`-less composite request is wrongly rejected
`4.00 Bad Request` — a real Leshan-interop bug.

**Read-Composite is best effort and non-atomic** (Core TS 1.2.2 §6.3.8): paths that have no valid
value, or whose Resource does not support Read, are **omitted from the response** — there is no
per-path error field. A composite read of three paths where one is missing returns values for two
and nothing for the third, with 2.05 Content — not an overall 4.04. Clients consuming the response
must therefore treat "absent" as "unavailable" rather than assuming positional alignment with the
request.

**Write-Composite is atomic** (Core TS 1.2.2 §6.3.9): the client MUST reject the whole request if it
cannot write every listed value — it checks existence, write access and writability of every
addressed Object/Resource before applying anything. Resources not listed are not affected.

**Observe-Composite / Cancel Observation-Composite** (Core TS 1.2.2 §6.4.4, §6.4.5): FETCH +
Observe:0 on `/` with the path list in the body; notifications include **every** listed path even if
only one met its condition. Observation attributes supplied with the request MUST be Object-level
ones (pmin/pmax/epmin/epmax/con). Cancel is FETCH + Observe:1 carrying exactly the same path list
(Transport TS 1.2.2 Table 6.4.5-1); if Observe-Composite is supported, Cancel MUST be.

## F. Create wire-format — the URI / payload / Location-Path triad

Create has three coupled wire details that must agree (Core TS 1.2.2 §6.3.6; Transport TS 1.2.2
§6.4.4):

1. **Request URI is always `POST /{ObjId}`** — the instance ID **never** goes in the
   URI, even when the server chooses it. The Object MUST be one the client announced at
   Register/Update.
2. **Payload** (TLV, LwM2M CBOR, SenML CBOR or SenML JSON) encodes who allocates the instance ID:
   - **Client-assigned instance ID:** send **bare Resource TLVs** with **no Object
     Instance TLV wrapper** (or an LwM2M CBOR payload without an Object Instance reference), so
     the client MUST allocate the ID itself (Core §6.3.6). Wrapping in an Object Instance TLV with
     id 0 *forces* instance 0; for a single-instance Object whose instance 0 already exists that is
     rejected `4.00 Bad Request` ("Target already exists", Transport Table 6.7-3). A single-instance
     Object MUST be given ID 0 on Create.
   - **Server-assigned instance ID:** **wrap** the Resources in an Object Instance TLV
     carrying the chosen ID.
3. **Success = `2.01 Created`** with the new location returned as **`Location-Path`** options.
   If any error occurs the client MUST NOT create anything (Core §6.3.6).

```
POST /11                           (Object URI only; never /11/2)
Content-Format: application/vnd.oma.lwm2m+tlv

-> 2.01 Created
   Location-Path: 11
   Location-Path: 2                (two options -> reassemble to /11/2)
```

The response carries **one `Location-Path` option per path segment** (e.g. `"11"` then `"2"` →
`/11/2`). Reassemble **all** segments in order; do not read only the last option. The same applies to
the registration location (`"rd"`, `"{id}"`), which must be under `/rd` (Transport TS 1.2.2 §6.4.3).

## G. Registration interface contract

Registration uses the `/rd` Resource Directory-style endpoints and CoRE Link Format (Core TS 1.2.2
§6.2; Transport TS 1.2.2 §6.4.3; RFC 6690):

| Operation | Method + URI | Notes |
|-----------|--------------|-------|
| Register | `POST /rd` + Uri-Query | Body = Object/Instance list in CoRE Link Format; → 2.01 Created, `Location-Path` options `rd`,`{id}` |
| Update | `POST /rd/{id}` [+ Uri-Query] | Carries **only** the parameters that changed since the last registration (Core §6.2.2); an empty Update just refreshes the lifetime |
| De-register | `DELETE /rd/{id}` | |

**Register query parameters (Core TS 1.2.2 Table 6.2.1-1; Transport TS 1.2.2 Table 6.4.3-1):**

| Param | Required | Meaning |
|-------|----------|---------|
| `ep` | No (v1.1+; **Yes** in v1.0, TS 1.0.2 §5.3.1) | Endpoint Client Name. The Register SHOULD include it; it MUST be present when the security protocol does not supply an authenticated identifier (Core §7.4). Unique per server. |
| `lt` | **Yes** | Lifetime in seconds; MUST equal /1/x/1 of the matching Server Object Instance. **No default value.** |
| `lwm2m` | **Yes** | Enabler version, e.g. `1.2`; MUST be an approved version number. |
| `b` | No (default `U`) | Binding: any combination of `U`, `M`, `H`, `T`, `S`, `N` (Core §6.2.1.2); SHOULD match /3/0/16. `UQ`/`SQ`/`UQS` are v1.0-only encodings. |
| `Q` | No | Queue Mode flag, v1.1+, no value. |
| `sms` | No | MSISDN / External Identifier for the SMS binding or SMS trigger. |
| `pid` | No (v1.2) | Profile ID — object list by reference; ASCII `oma:…`, `v:…` or `<hash-suite>:<hex>`; may repeat and may be mixed with a payload list. Server support is optional; an unresolved pid → error (4.09) and the client MUST re-register with the full list. |

Example: `POST /rd?ep=node&lt=300&lwm2m=1.2&b=U&Q` with payload `</1/0>,</3/0>,</4/0>,</5>`.
Security Object /0, OSCORE /21 and COSE /23 MUST NOT appear in the payload list (Core Table 6.2.1-1).

**Link-format parsing must be defensive:** it must tolerate a root `</>` entry (used for
`ct=` and `rt="oma.lwm2m"` alternate-path advertisement) and non-numeric path segments **without
throwing** — malformed or unexpected link-format is an availability risk, not merely a data-quality
one. Unknown link parameters MUST NOT be treated as errors (Core §6.2.1). See
`registry-ddf-authoring.md` for the link-format structure derived from object definitions.

**Version compatibility (Core TS 1.2.2 §5.1 and §6.2.1):** a server MUST support clients of the same
major.minor version and SHOULD support all versions of the same major version in both directions. A
server that does not support the client's `lwm2m` version MUST refuse the registration — `4.12
Precondition Failed` (Transport TS 1.2.2 Table 6.7-2).

> **Implementation guidance (not normative).** A client registering with `lwm2m=1.2` may retry as
> `lwm2m=1.1` not only on `4.12` but also on REGISTER retransmit exhaustion / silence, because some
> servers blackhole a version they do not understand rather than replying 4.12. Perform **exactly
> one** 1.1 retry before hard failure — do not loop. No OMA specification defines this retry.

## H. Write-Attributes

Write-Attributes is a **PUT with `Uri-Query` parameters** (no payload) that sets NOTIFICATION-class
attributes at Object, Object Instance, Resource or Resource Instance level (Core TS 1.2.2 §6.3.4;
attribute definitions in §7.3.2 Table 7.3.2-1; CoAP mapping in Transport TS 1.2.2 Table 6.4.4-1).
Only NOTIFICATION-class attributes are writable; a parameter given without a value **unsets** that
attribute at that level.

| Attribute | Meaning | Attachment (Table 7.3.2-1) | Since |
|-----------|---------|----------------------------|-------|
| `pmin` | Minimum period (s); default = /1/x/2 else 0 | O / OI / R / RI | v1.0 |
| `pmax` | Maximum period (s); 0 or `< pmin` → ignored | O / OI / R / RI | v1.0 |
| `gt` | Greater-than threshold | R / RI, numeric readable resources only | v1.0 |
| `lt` | Less-than threshold | R / RI, numeric readable resources only | v1.0 |
| `st` | Step (minimum change) | R / RI, numeric readable resources only | v1.0 |
| `epmin` | Min evaluation period (s) | O / OI / R / RI | **v1.1** |
| `epmax` | Max evaluation period (s) | O / OI / R / RI | **v1.1** |
| `edge` | `0` = falling, `1` = rising edge of a Boolean resource | R / RI, Boolean readable | **v1.2** |
| `con` | `0`/`1`; `1` = notifications MUST be confirmable | O / OI / R / RI | **v1.2** |
| `hqmax` | Bound on the offline historical queue; effective only when /1/x/6 Notification Storing is true | O / OI / R / RI | **v1.2** |

(`O` = Object, `OI` = Object Instance, `R` = Resource, `RI` = Resource Instance.) Version
attribution: epmin/epmax appear in Core TS 1.1 §5.1.2; edge, confirmable notification and maximum
historical queue are listed as new attributes in Core TS 1.2 (change history).

Attributes written with Write-Attributes are permanent and apply to every later Observe that does
not supply its own observation attributes (Core §6.4.1 note), so they **must be storable even when
no observation exists yet** — store and return `2.04 Changed`.

> **Implementation guidance (not normative).** If the device genuinely cannot record the attribute
> (resource exhaustion), answer with a 5.xx server error per the generic rule in Transport TS 1.2.2
> §6.7, **not** `4.04 Not Found` — the target exists.

**Validation invariants (Core TS 1.2.2 §7.3.2, §6.4.2):**
- `lt < gt` and `lt + 2·st < gt`. A Write-Attributes, Observe or Observe-Composite violating either
  MUST be rejected — `4.00 Bad Request` ("attributes … MUST be consistent … otherwise rejected
  (4.00)", Transport TS 1.2.2 §6.4.4). The relation is strict (`<`), not `≤`.
- `epmin < epmax` when both are set.
- `pmax` smaller than `pmin` is not an error: pmax is ignored for that observation. `pmax = 0` also
  means ignored.
- `st` applies as "change since last notification ≥ st"; the spec states no explicit `st > 0` rule,
  but a zero step degenerates to "notify on any change" (unverified against OMA spec library beyond
  Table 7.3.2-1).

Inheritance and precedence (Core §7.3.2): attributes apply to the attached level and below; a
lower-level attachment wins over an upper one.

## I. Send

Send **(v1.1+)** is a **client-initiated** `POST /dp` carrying a SenML or LwM2M CBOR payload.
Introduced in Core TS 1.1 as §6.3.10 (DM&SE interface); moved to the Information Reporting
interface as Core TS 1.2 §6.4.6 (same number in 1.2.2). CoAP mapping: Transport TS 1.2.2 §6.4.5 and
Table 6.4.5-1. Client and server MAY support it.

- Content-Format MUST be LwM2M CBOR (11544), SenML CBOR (112) or SenML JSON (110). SenML CBOR is the
  usual default on constrained links, but the spec mandates no single default.
- The client is identified by its **security association / peer address**, not by a URL
  endpoint in the request.
- The client may only report Object Instances it has registered with that server, and only
  Resources that server may Read (Core §6.4.6); unknown optional Resources MUST NOT be treated as an
  error by the server.
- Response codes (Transport TS 1.2.2 Table 6.7-4): `2.04 Changed`; `4.00 Bad Request`;
  `4.04 Not Found` when a reported Object was not registered to the server. These are the only
  codes defined for Send.
- **Mute Send** /1/x/23 (Boolean, RW, Optional — OMA-SUP-XML_1 v1.2): **true or absent ⇒ Send is
  disabled**; `false` enables it.
- Send MAY be sent as a Non-Confirmable message over UDP (Transport TS 1.2.2 §6.8.1).
- For **queue-mode** clients, receiving a Send is a **strong wake indicator** — the
  server may use it to flush any queued downlink requests (implementation guidance; the normative
  wake signal is the Registration Update, Transport §6.5).

> **Unsourced claim removed.** Earlier text asserted that a server MUST reject a Send from a client
> registered as 1.0 with `4.05 Method Not Allowed`. No OMA specification states this; a 1.0 client
> simply has no Send operation, and Send's defined codes are 2.04 / 4.00 / 4.04 only. Treat any such
> rejection as an implementation choice.

```
POST /dp                           (client -> server)
Content-Format: application/senml+cbor
Body (SenML): [ {"bn":"/3303/0/", "n":"5700", "v":21.4} ]
-> 2.04 Changed
```

## J. Minimal-client conformance — always answer, even to refuse

Even a stripped-down minimal client **must answer every inbound method definitively**.
The guiding rule is *"always answer, even to refuse"* — a silent drop makes the server
retransmit until timeout, which looks like a dead device.

| Situation | Required response | Citation |
|-----------|-------------------|----------|
| Write / Execute / Create / Delete on a target that does not allow it | `4.05 Method Not Allowed` | Core TS 1.2.2 §6.3.3, §6.3.5, §6.3.6; Transport TS 1.2.2 Table 6.7-3 ("Target is not allowed for … operation"); RFC 7252 §5.9.2.6 |
| Read / Discover / Write of a missing target | `4.04 Not Found` | Transport TS 1.2.2 Table 6.7-3 ("URI of … operation is not found") |
| Operation the server lacks ACL rights for | `4.01 Unauthorized` | Core TS 1.2.2 §8.2 Table 8.2-1; Transport TS 1.2.2 Table 6.7-3 ("Access Right Permission Denied") |
| Failure not describable by any 4.xx | 5.xx | Transport TS 1.2.2 §6.7 (rule after Table 6.7-4) |
| Any inbound request | never silently drop | — |

## K. Conformance response-code quick table

Derived from Transport TS 1.2.2 §6.7 Tables 6.7-2 to 6.7-4 and Core TS 1.2.2 §6.3.5 / §8.2; the OMA
ETS (`OMA-ETS-LightweightM2M_INT`) exercises these interoperably:

| Stimulus | Response | Notes |
|----------|----------|-------|
| Execute on an executable resource | `2.04 Changed` | e.g. Reboot `/3/0/4` must then re-register with a fresh `POST /rd` |
| Execute on a non-Executable resource, Object Instance or Resource Instance | `4.05 Method Not Allowed` | Core §6.3.5 |
| Execute with an argument the client does not understand | `4.00 Bad Request` | Transport Table 6.7-3; lenient ignoring is an implementation choice |
| Write to a read-only resource | `4.05 Method Not Allowed` | Core §6.3.3 |
| Write with wrong type / out-of-range / bad Objlnk | `4.00 Bad Request` | Core §6.3.3 |
| Write/Create with unsupported Content-Format | `4.15 Unsupported Content-Format` | Transport §6.4.4 |
| Read with an Accept the client cannot satisfy | `4.06 Not Acceptable` | Transport §6.4.4 |
| Any DM&SE/IR operation without ACL rights | `4.01 Unauthorized` | "Access Right Permission Denied" |
| Read of a non-existent target | `4.04 Not Found` | |
| Register | `2.01 Created` | |
| Register with missing `lt`/`lwm2m` or unknown parameter | `4.00 Bad Request` | Transport Table 6.7-2 |
| Register for a disallowed endpoint name | `4.03 Forbidden` | Register only |
| Register with an unresolvable object list / pid | `4.09 Conflict` | Register only |
| Register with an unsupported LwM2M version | `4.12 Precondition Failed` | Register only (Core §6.2.1 MUST refuse) |
| Read | `2.05 Content` | |
| Write / Discover | `2.04 Changed` / `2.05 Content` | Write → 2.04; Discover → 2.05 |
| Send with an unregistered Object | `4.04 Not Found` | Transport Table 6.7-4 |

## Key specification references

| Reference | Section | Topic |
|-----------|---------|-------|
| Core TS 1.2.2 | §5.1 | Server/client version compatibility |
| Core TS 1.2.2 | §6.2.1, §6.2.2, §6.2.3 | Register (Table 6.2.1-1), Update, De-register |
| Core TS 1.2.2 | §6.2.1.2 | Binding values U/M/H/T/S/N |
| Core TS 1.2.2 | §6.3.1 | Read |
| Core TS 1.2.2 | §6.3.2 | Discover (depth, Tables 6.3.2-1/-2) |
| Core TS 1.2.2 | §6.3.3 | Write (Replace / Partial Update) |
| Core TS 1.2.2 | §6.3.4 | Write-Attributes |
| Core TS 1.2.2 | §6.3.5 | Execute (argument ABNF) |
| Core TS 1.2.2 | §6.3.6 | Create |
| Core TS 1.2.2 | §6.3.7 | Delete |
| Core TS 1.2.2 | §6.3.8 | Read-Composite (best effort) |
| Core TS 1.2.2 | §6.3.9 | Write-Composite (atomic) |
| Core TS 1.2.2 | §6.4.1, §6.4.2, §6.4.3 | Observe, Notify, Cancel Observation |
| Core TS 1.2.2 | §6.4.4, §6.4.5 | Observe-Composite, Cancel Observation-Composite |
| Core TS 1.2.2 | §6.4.6 | Send (v1.1: Core TS 1.1 §6.3.10) |
| Core TS 1.2.2 | §7.3, §7.3.2 | Attributes; NOTIFICATION class (Table 7.3.2-1) |
| Core TS 1.2.2 | §7.5 | Data formats and Content-Format codes (Tables 7.5-1 to 7.5-3) |
| Core TS 1.2.2 | §8, §8.2 | Access control; authorisation (Table 8.2-1) |
| Transport TS 1.2.2 | §6.4.3 | Registration mapping (Tables 6.4.3-1/-2) |
| Transport TS 1.2.2 | §6.4.4 | DM&SE mapping and admissible formats (Table 6.4.4-1) |
| Transport TS 1.2.2 | §6.4.5 | Information Reporting mapping; Observe option values (Table 6.4.5-1) |
| Transport TS 1.2.2 | §6.7 | Response codes (Tables 6.7-1 to 6.7-4; generic 5.xx rule) |
| `OMA-ETS-LightweightM2M_INT` | — | Interoperability test cases |
| `RFC 7252` | §5.8, §5.9 | CoAP methods and response codes |
| `RFC 8132` | — | FETCH / PATCH / iPATCH for CoAP |
| `RFC 7641` | §3.1 | CoAP Observe option |
| `RFC 8790` | — | SenML Fetch/Patch records (composite path lists) |
| `RFC 6690` | — | CoRE Link Format |

## Related files

- [`protocol-details.md`](protocol-details.md) — interface→method table, transport
  bindings, and content-format codes.
- [`registry-ddf-authoring.md`](registry-ddf-authoring.md) — object/DDF definitions that
  drive link-format output and Resource type/operation metadata.
- [`objects.md`](objects.md) — Object and Resource models, declared data types referenced
  by Plain Text re-parsing and executable-resource determination.
