# Operation-to-CoAP mapping

This reference explains *how* the LwM2M interface operations map onto CoAP methods,
options, and URI levels — and where that mapping is overloaded, subtle, or a frequent
source of interoperability bugs. The companion file `protocol-details.md` carries the
interface→method *table*; this file fills in the *semantics* behind each row, the
wire-format details, and the exact conformance response codes.

The recurring theme is that several LwM2M operations share a single CoAP method and are
distinguished only by **target-path depth**, **CoAP options**, or **request body** —
not by the method verb alone. A correct dispatcher therefore cannot branch on the CoAP
method in isolation.

## Table of contents

- [Master mapping table](#master-mapping-table)
- [A. POST is overloaded — disambiguate by target-path depth](#a-post-is-overloaded--disambiguate-by-target-path-depth)
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
overloaded methods. Read it together with [§A](#a-post-is-overloaded--disambiguate-by-target-path-depth)
and [§D](#d-get-is-overloaded--disambiguate-by-coap-options).

| Operation | CoAP method | Disambiguator | URI level | Success code | Notes |
|-----------|-------------|---------------|-----------|--------------|-------|
| Read | GET | no `Accept:40`, no Observe | `/o/i`, `/o/i/r`, `/o` | 2.05 Content | Returns TLV/SenML values |
| Discover | GET | `Accept: application/link-format` (CT 40) | `/o`, `/o/i`, `/o/i/r` | 2.05 Content | Returns CoRE Links, not values |
| Observe (register) | GET | Observe option = 0 | `/o`, `/o/i`, `/o/i/r` | 2.05 Content | Notifications follow |
| Observe (cancel, active) | GET | Observe option = 1 | same as registration | 2.05 Content | Active cancel |
| Write (Replace) | PUT | — | `/o/i` | 2.04 Changed | Omitted Resources reset |
| Write (Partial Update) | POST | TLV/SenML body | `/o/i` | 2.04 Changed | Omitted Resources untouched |
| Write (single) | PUT or POST | — | `/o/i/r` | 2.04 Changed | Single-value formats allowed |
| Write-Attributes | PUT | Uri-Query params | `/o`, `/o/i`, `/o/i/r` | 2.04 Changed | pmin/pmax/gt/lt/st/… |
| Execute | POST | target Resource is Executable | `/o/i/r` | 2.04 Changed | Optional text args |
| Create | POST | bare Object URI | `/o` | 2.01 Created | Location-Path → new instance |
| Delete | DELETE | — | `/o/i` | 2.02 Deleted | |
| Read-Composite (v1.2+) | FETCH | body-addressed | `/` (root) | 2.05 Content | Paths in SenML body |
| Write-Composite (v1.2+) | iPATCH | body-addressed | `/` (root) | 2.04 Changed | Paths in SenML body |
| Register | POST | — | `/rd` + Uri-Query | 2.01 Created | Location-Path `/rd/{id}` |
| Update | POST | — | `/rd/{id}` | 2.04 Changed | |
| De-register | DELETE | — | `/rd/{id}` | 2.02 Deleted | |
| Send (v1.1+) | POST | — | `/dp` | 2.04 Changed | Client-initiated, SenML body |

`o` = Object ID, `i` = Object Instance ID, `r` = Resource ID throughout.

## A. POST is overloaded — disambiguate by target-path depth

POST is the most overloaded method in LwM2M. The same verb means three different
operations depending solely on the depth of the target URI. This is the single most
common LwM2M↔CoAP dispatch bug.

```
POST /{ObjId}                  -> Create            (bare Object URI)
POST /{ObjId}/{InstId}         -> Write (Partial)   (Object Instance URI, TLV/SenML body)
POST /{ObjId}/{InstId}/{ResId} -> Execute           (Resource URI, executable resource)
```

- **POST on a bare Object URI** `/{ObjId}` = **Create**
  (`OMA-TS-LightweightM2M_Core-V1_2_2, §5.4.6`; `TODO(verify): exact subsection — one
  source cites §6.3.3, a matrix labelled §6.3.6`).
- **POST on an Object Instance URI** `/{ObjId}/{InstId}` with a TLV/SenML body =
  **Write (Partial Update)** (`OMA-TS-LightweightM2M_Core-V1_2_2, §5.4.3`): only the
  Resources carried in the body are written; **omitted Resources are left unchanged.**
- **POST on a Resource URI** `/{ObjId}/{InstId}/{ResId}` that is Executable =
  **Execute** (`OMA-TS-LightweightM2M_Core-V1_2_2, §5.4.5`), with optional text
  arguments in the body.

A dispatcher must therefore branch a POST **three ways by URI depth** before doing
anything else. A real, fixed bug: routing *every* instance-scoped POST to Create makes
a client wrongly return `4.05 Method Not Allowed` to legitimate Partial-Update writes,
because Create is not valid against an already-existing Object Instance URI.

## B. PUT vs POST write semantics, and a correctness trap

PUT and POST both perform a Write against an Object Instance, but with **opposite
semantics for omitted Resources** (`OMA-TS-LightweightM2M_Core-V1_2_2, §5.4.3`):

- **PUT on an Object Instance** = **Write (Replace)**: Resources *not* present in the
  payload **must be reset** to their default/initial values (or removed if optional).
  The instance after the write reflects only what the payload described.
- **POST on an Object Instance** = **Write (Partial Update)**: Resources *not* present
  in the payload are **left untouched**.

```
Before:  /3/0 = { 0:"Acme", 1:"M-1", 13:1700000000 }

PUT  /3/0  body={ 0:"Acme" }    -> Replace -> { 0:"Acme" }  (1,13 reset/removed)
POST /3/0  body={ 0:"Acme" }    -> Partial -> { 0:"Acme", 1:"M-1", 13:1700000000 }
```

**Common bug:** routing both PUT and POST into a single writer that only applies the
carried Resources makes Replace silently behave like Partial Update — omitted Resources
are wrongly preserved. Flag this whenever PUT and POST share a write code path: the
Replace branch must additionally reset/remove every Resource absent from the payload.

## C. Write scope and content-format admissibility by URI depth

A Write must address **at least an Object Instance**. A Write targeting a bare Object
URI `/{ObjId}` is out of scope and must be rejected `4.05 Method Not Allowed`.

| Write target | URI | Admissible content-formats |
|--------------|-----|----------------------------|
| Resource-level | `/o/i/r` | Plain Text (CT 0), TLV, SenML JSON, SenML CBOR |
| Instance-level | `/o/i` | TLV, SenML JSON, SenML CBOR (multi-resource only) |
| Object-level | `/o` | — (not a valid Write target → 4.05) |

Plain Text is admissible only at the Resource level because it carries **no type tag**:
a Plain Text value must be re-parsed into the Resource's *declared* data type from the
Object definition (see `objects.md`). TLV and SenML are **self-describing** — each item
carries its identifier and value encoding — so they are admissible wherever multiple
Resources may appear. Instance-level writes therefore cannot use Plain Text.

## D. GET is overloaded — disambiguate by CoAP options

GET maps to three operations, distinguished by CoAP **options**, never by the verb:

```
GET, Accept: application/link-format (CT 40)  -> Discover  (returns links)
GET, Observe option present                   -> Observe   (0=register, 1=cancel)
GET, neither of the above                     -> Read      (returns values)
```

- **GET with `Accept: application/link-format`** (CT 40) = **Discover**
  (`OMA-TS-LightweightM2M_Core-V1_2_2, §5.4.4`): returns the resource/attribute link
  list in CoRE Link Format (`RFC 6690`), **not** Resource values.
- **GET carrying a CoAP Observe option** (`RFC 7641 §3.1`) = **Observe**: Observe
  option value **0 registers** an observation, **1 cancels** it (active cancellation)
  (`OMA-TS-LightweightM2M_Core-V1_2_2, §3.6`).
- **Plain GET** — no `Accept: 40` and no Observe option — = **Read**
  (`OMA-TS-LightweightM2M_Core-V1_2_2, §5.4.1`): returns TLV/SenML values.

Note that Discover and Observe can each apply at Object, Object Instance, or Resource
level, so URI depth does **not** disambiguate GET — only the options do.

## E. Composite operations are body-addressed

Composite operations **(v1.2+)** break the usual rule that the target lives in the URI.

- **Read-Composite** = CoAP **FETCH**; **Write-Composite** = CoAP **iPATCH**
  (`RFC 8132`; `OMA-TS-LightweightM2M_Core-V1_2_2, §5.4.8/§5.4.9` —
  `TODO(verify): exact subsections`).
- Both address the **root** `/` and carry their **target resource paths inside the
  SenML request body**, not in `Uri-Path`.

```
FETCH /              (no Uri-Path)
Content-Format: application/senml+cbor
Body (SenML): [ {"n":"/3/0/0"}, {"n":"/3/0/1"}, {"n":"/6/0/0"} ]
```

A dispatcher must route **FETCH/iPATCH before parsing the URI**. If it parses the URI
first, a deliberately `Uri-Path`-less composite request is wrongly rejected
`4.00 Bad Request` — a real Leshan-interop bug.

Composite reads carry a **per-path partial-failure model**: in the response, successful
paths carry value fields, while failed paths carry an `error` field (and no value) at
that path. Clients **must surface per-path failures** rather than treating the whole
request as a single success/fail. A composite read of three paths where one is missing
returns values for two and an `error` entry for the third — not an overall 4.04.

## F. Create wire-format — the URI / payload / Location-Path triad

Create has three coupled wire details that must agree (`OMA-TS-LightweightM2M_Core-V1_2_2,
§5.4.6`):

1. **Request URI is always `POST /{ObjId}`** — the instance ID **never** goes in the
   URI, even when the server chooses it.
2. **Payload** encodes who allocates the instance ID:
   - **Client-assigned instance ID:** send **bare Resource TLVs** with **no Object
     Instance TLV wrapper**, so the client allocates the ID itself.
     Wrapping in an Object Instance TLV with id 0 *forces* instance 0; for a
     single-instance object whose instance 0 already exists, that is rejected
     `4.00 Bad Request`.
   - **Server-assigned instance ID:** **wrap** the Resources in an Object Instance TLV
     carrying the chosen ID.
3. **Success = `2.01 Created`** with the new location returned as **`Location-Path`**.

```
POST /11                           (Object URI only; never /11/2)
Content-Format: application/vnd.oma.lwm2m+tlv

-> 2.01 Created
   Location-Path: 11
   Location-Path: 2                (two options -> reassemble to /11/2)
```

The response often carries **multiple `Location-Path` options** (one per path segment,
e.g. `"11"` then `"2"` → `/11/2`). Reassemble **all** segments in order; do not read
only the last option.

## G. Registration interface contract

Registration uses the Resource Directory endpoints and CoRE Link Format
(`OMA-TS-LightweightM2M_Core-V1_2_2, §6.2`; `RFC 6690`):

| Operation | Method + URI | Notes |
|-----------|--------------|-------|
| Register | `POST /rd` + Uri-Query | Body = object list in CoRE Link Format; → 2.01 Created, `Location-Path: /rd/{id}` |
| Update | `POST /rd/{id}` | Refresh lifetime / object list |
| De-register | `DELETE /rd/{id}` | |

**Register query parameters:** `ep` (endpoint name, **required**), `lt` (lifetime in
seconds), `lwm2m` (version), `b` (binding: `U`/`T`/`S`/`UQ`/`SQ`).

**Link-format parsing must be defensive:** it must tolerate a root `</>` entry and
non-numeric path segments **without throwing** — malformed or unexpected link-format is
an availability risk, not merely a data-quality one. See `registry-ddf-authoring.md`
for the link-format structure derived from object definitions.

**Version fallback (`OMA-TS-LightweightM2M_Core-V1_2_2, §6.2.5`):** a client registering
with `lwm2m=1.2` should retry as `lwm2m=1.1` **not only on `4.12 Precondition Failed`**
but **also on REGISTER retransmit exhaustion / silence**. A server that does not
understand 1.2 may **blackhole** the request rather than reply 4.12. Perform **exactly
one** 1.1 retry before hard failure — do not loop.

## H. Write-Attributes

Write-Attributes is a **PUT with `Uri-Query` parameters** (no payload) that sets
notification-class attributes (`OMA-TS-LightweightM2M_Core-V1_2_2, §6.6.3`; NOTIFICATION
class, §6.4.x):

| Attribute | Meaning | Notes |
|-----------|---------|-------|
| `pmin` | Minimum period (s) | |
| `pmax` | Maximum period (s) | |
| `gt` | Greater-than threshold | numeric resources only |
| `lt` | Less-than threshold | numeric resources only |
| `st` | Step | numeric resources only |
| `epmin` **(v1.1+)** | Min evaluation period | |
| `epmax` **(v1.1+)** | Max evaluation period | |
| `con` **(v1.1+)** | Confirmable notification control | |
| `hqmax` | Bound on offline historical queue | |

Attributes **must be storable even when no observation exists yet** — store and return
`2.04 Changed`, holding the attributes for a future Observe. A storage failure is a
resource-limit **`5.00 Internal Server Error`**, **not** `4.04 Not Found` (the target
exists; the device simply could not record the attribute).

**Validation invariants:** `pmin ≤ pmax`, `epmin ≤ epmax`, `st > 0`.
`TODO(verify): exact gt/lt relational requirement — implementations differ on whether
gt > lt or gt ≥ lt is required.`

## I. Send

Send **(v1.1+)** is a **client-initiated** `POST /dp` carrying a SenML payload, default
**SenML-CBOR (CT 112)** (`OMA-TS-LightweightM2M_Core-V1_2_2, §6.4.6/§6.5`):

- The client is identified by its **security association / peer address**, not by a URL
  endpoint in the request.
- A server **MUST reject** a Send from a client registered as **1.0** with
  `4.05 Method Not Allowed` (Send did not exist in 1.0).
- For **queue-mode** clients, receiving a Send is a **strong wake indicator** — the
  server may use it to flush any queued downlink requests.

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
| Unsupported WRITE/CREATE/DELETE/EXECUTE | `4.05 Method Not Allowed` | `RFC 7252 §5.9.1.1`; `OMA-TS-LightweightM2M_Core-V1_2_2, §5.4` |
| Read of a missing target | `4.04 Not Found` | `OMA-TS-LightweightM2M_Core-V1_2_2, §5.4` |
| Any inbound request | never silently drop | — |

## K. Conformance response-code quick table

From the OMA ETS interop fixtures (cite `OMA-TS-LightweightM2M_Core-V1_2_2, §5.4.5/§6.4.5`
and `OMA-ETS-LightweightM2M_INT`):

| Stimulus | Response | Notes |
|----------|----------|-------|
| Execute on an executable resource | `2.04 Changed` | e.g. Reboot `/3/0/4` must then re-register with a fresh `POST /rd` |
| Execute on a non-Executable resource | `4.05 Method Not Allowed` | |
| Execute with unexpected args | `4.00 Bad Request` (strict) or ignore (lenient) | implementation choice |
| Write to a read-only resource | `4.05 Method Not Allowed` | |
| Read of a non-existent target | `4.04 Not Found` | |
| Register | `2.01 Created` | |
| Read | `2.05 Content` | |
| Write / Discover | `2.04 Changed` / `2.05 Content` | Write → 2.04; Discover → 2.05 |

## Key specification references

| Reference | Section | Topic |
|-----------|---------|-------|
| `OMA-TS-LightweightM2M_Core-V1_2_2` | §5.4.1 | Read |
| `OMA-TS-LightweightM2M_Core-V1_2_2` | §5.4.3 | Write (Replace / Partial Update) |
| `OMA-TS-LightweightM2M_Core-V1_2_2` | §5.4.4 | Discover |
| `OMA-TS-LightweightM2M_Core-V1_2_2` | §5.4.5 | Execute |
| `OMA-TS-LightweightM2M_Core-V1_2_2` | §5.4.6 | Create / Delete `TODO(verify): Create subsection (§6.3.3 vs §6.3.6 in matrices)` |
| `OMA-TS-LightweightM2M_Core-V1_2_2` | §5.4.8/§5.4.9 | Composite ops `TODO(verify): exact subsections` |
| `OMA-TS-LightweightM2M_Core-V1_2_2` | §3.6 | Observe register/cancel option values |
| `OMA-TS-LightweightM2M_Core-V1_2_2` | §6.2 / §6.2.5 | Registration; version fallback |
| `OMA-TS-LightweightM2M_Core-V1_2_2` | §6.4.6 / §6.5 | Send |
| `OMA-TS-LightweightM2M_Core-V1_2_2` | §6.6.3 / §6.4.x | Write-Attributes / NOTIFICATION class |
| `OMA-ETS-LightweightM2M_INT` | — | Interop conformance fixtures |
| `RFC 7252` | §5.9, §5.9.1.1 | CoAP response codes; 4.05 |
| `RFC 8132` | — | FETCH / PATCH / iPATCH for CoAP |
| `RFC 7641` | §3.1 | CoAP Observe option |
| `RFC 6690` | — | CoRE Link Format |

## Related files

- [`protocol-details.md`](protocol-details.md) — interface→method table, transport
  bindings, and content-format codes.
- [`registry-ddf-authoring.md`](registry-ddf-authoring.md) — object/DDF definitions that
  drive link-format output and Resource type/operation metadata.
- [`objects.md`](objects.md) — Object and Resource models, declared data types referenced
  by Plain Text re-parsing and executable-resource determination.
