# Northbound Interface — REST Mapping, Error Model, Auth/RBAC, and Observe Streaming

How a LwM2M server exposes its managed device fleet to operators and northbound applications through an HTTP/REST northbound interface (NBI): the verb→operation mapping, the problem-details error model, OAuth 2.1 + DPoP authentication, server-owned RBAC and tenant isolation, and the Observe→event-streaming bridge. This aligns with the direction of the OMA LwM2M v2.0 Northbound API companion specification. Where content is design recommendation rather than normative protocol, it is marked **implementation pattern**.

## Table of Contents
1. [REST verb to LwM2M operation mapping](#rest-verb-to-lwm2m-operation-mapping)
2. [Error model — problem details](#error-model--problem-details)
3. [Create and observe response convention](#create-and-observe-response-convention)
4. [Authentication — OAuth 2.1 and DPoP](#authentication--oauth-21-and-dpop)
5. [Authorisation — server-owned RBAC and tenant isolation](#authorisation--server-owned-rbac-and-tenant-isolation)
6. [Observe to northbound streaming](#observe-to-northbound-streaming)
7. [Event envelope and taxonomy](#event-envelope-and-taxonomy)
8. [SSE delivery mechanics for browser clients](#sse-delivery-mechanics-for-browser-clients)
9. [Content negotiation for browsers](#content-negotiation-for-browsers)
10. [EST credential surface](#est-credential-surface)
11. [API strategy and migration](#api-strategy-and-migration)
12. [Key specification references](#key-specification-references)
13. [Related files](#related-files)

---

## REST verb to LwM2M operation mapping

A clean NBI maps HTTP verbs onto LwM2M device-management operations under a tenant-scoped resource URI. This is a recommended NBI design (**implementation pattern**), not a normative mapping mandated by `OMA-TS-LightweightM2M_Core`.

URI template:

```
/v1/tenants/{tenant}/devices/{endpoint}/{obj}/{inst}/{res}
```

| HTTP verb | LwM2M operation(s) |
|-----------|--------------------|
| `GET`     | Read |
| `PUT`     | Write (Replace) |
| `PATCH`   | Write (Partial Update) |
| `POST`    | Execute / Create / Observe |
| `DELETE`  | Delete / Cancel-Observe |

A `?discover=true` query toggle selects Discover instead of a plain instance Read.

Crucially, a plain object-**instance Read** and **Discover** are distinct LwM2M operations and must not be collapsed:

- Instance Read aggregates the child resource **values** of the instance.
- Discover returns the resource **tree plus attached attributes** (notification/observation attributes), and returns **no values**.

A server that implements only Discover and returns `501 Not Implemented` on a plain instance Read is **incomplete** — the two operations serve different operator needs and both must be exposed.

---

## Error model — problem details

Errors are surfaced as `RFC 9457` Problem Details for HTTP APIs. The standard members are used as defined:

| Member     | Meaning |
|------------|---------|
| `type`     | URI reference identifying the problem type |
| `title`    | Short, human-readable summary |
| `status`   | HTTP status code (numeric) |
| `detail`   | Human-readable explanation specific to this occurrence |
| `instance` | URI reference identifying this specific occurrence |

The document is **extended** with a namespaced member, `lwm2m:coapCode`, that carries the underlying CoAP response code through to the HTTP layer (for example `"4.04 Not Found"`). This gives operators both the HTTP view and the CoAP view of the same failure.

```json
{
  "type": "https://errors.example/lwm2m/not-found",
  "title": "Resource not found on device",
  "status": 404,
  "detail": "Object instance /3/0 is not present on endpoint dev-01",
  "instance": "/v1/tenants/acme/devices/dev-01/3/0",
  "lwm2m:coapCode": "4.04 Not Found"
}
```

Detection rule (**implementation pattern**): treat a body as a problem document when it contains `type` **and** `title` **and** a numeric `status`. Do not rely on the `Content-Type` alone.

**Version preconditions** surface as `412 Precondition Failed`, meaning "the device's LwM2M version does not support this operation." Composite operations require LwM2M v1.2+; Send and Composite operations are gated on the device's negotiated version. See [versions.md](versions.md) for the per-version capability matrix.

---

## Create and observe response convention

**Implementation pattern.** Creation-style operations (Create object instance, establish Observe) respond with:

- `201 Created`
- An **empty body**
- The new resource identifier — the instance id or observation id — as the **trailing segment** of the `Location` header.

```
HTTP/1.1 201 Created
Location: /v1/tenants/acme/devices/dev-01/observations/obs-7f3a
```

Client obligation: read the **full** response (do not short-circuit on an empty body), then extract and validate the tail of `Location` to obtain the identifier. A client that ignores `Location` on an empty-body 201 loses the only handle it has to the created instance or observation (and therefore cannot later cancel the observation).

---

## Authentication — OAuth 2.1 and DPoP

The standardised NBI authenticates requests with OAuth 2.1 plus sender-constrained tokens via DPoP (`RFC 9449`).

Request shape:

```
Authorization: DPoP <access_token>
DPoP: <dpop_proof_jwt>
```

The DPoP proof is a JWT with header `typ: "dpop+jwt"`, signed with `ES256` over a `P-256` key. Proof correctness requirements:

| Field / rule | Requirement |
|--------------|-------------|
| `htu`        | HTTP target URI with **query and fragment stripped** (`RFC 9449` §4.2) |
| `jti`        | Per-request unique nonce — replay defence |
| `ath`        | `base64url(SHA-256(access_token))` — binds the proof to the token (`RFC 9449` §4.3) |
| ECDSA signature | Raw `r‖s`, 64 bytes — **not** DER-encoded |
| `jkt` (thumbprint) | `RFC 7638` JWK thumbprint over the **lexicographically-ordered required JWK members** |
| `cnf.jkt`    | Present in the access token; binds proof-of-possession to the key |

### Security trap — empty-key verification (real bug)

A DPoP validator that **parses** the embedded JWK but then verifies the signature against an **empty key** (an always-success no-op) silently disables token-theft protection: a stolen access token becomes usable without a valid proof. The DPoP proof signature **MUST** actually be verified against the embedded JWK.

The symmetric failure exists on the client side: a proof generator must **sign asynchronously** — Web Crypto `SubtleCrypto` is async — and a synchronous code path silently emits an **empty proof**. Both ends require positive verification that signing and verification actually executed; "the JWK parsed" is not evidence that the signature was checked. See [security.md](security.md) for the broader credential-handling discipline.

---

## Authorisation — server-owned RBAC and tenant isolation

### Scope and tenant match are not authorisation

An OAuth scope check plus a tenant-match check is **authentication and routing**, not authorisation. Recommended **hybrid identity** model (**implementation pattern**):

- Keep the external OAuth IdP for **authentication** (who is the caller).
- Have the **server itself own** roles, permissions, and ACLs for **authorising** device-mutating operations (write / execute / delete).

This northbound, operator-facing RBAC is **distinct** from the LwM2M **Access Control Object** (Object `/2`), which governs device-side resource access between LwM2M servers and the client. Do not conflate the two — the Access Control Object does not authorise an operator dashboard, and operator RBAC does not control on-device access. See [objects.md](objects.md) for the Access Control Object semantics.

### Tenant isolation is a data-plane invariant

Tenant isolation **MUST** be enforced in the data plane, not merely at the identity edge. The following is a P0 data-leak pattern (real bug): authenticate the tenant from a JWT claim or a URI segment, then return the **global** device list. Every device-scoped read **MUST** apply a tenant→endpoint ownership filter so that a tenant can only ever observe endpoints it owns.

```
caller authenticated as tenant=acme
  GET /v1/tenants/acme/devices
  -> MUST return only endpoints owned by acme   (apply ownership filter)
  -> MUST NOT return the global device registry  (P0 leak if it does)
```

### Dev-posture cautions

- **Auth disabled when no JWKS configured.** A server may, when no JWKS is configured, disable auth entirely and treat every request as a default/anonymous tenant. This is acceptable for local development and **dangerous if shipped**. Guard against this posture reaching production.
- **"Configured ≠ functional" for DTLS.** A server can have PSK material seeded and Security Objects populated yet **never actually perform a DTLS handshake**. Verify the handshake path end-to-end rather than inferring security from configuration state. See [protocol-details.md](protocol-details.md) for the transport-security path.

---

## Observe to northbound streaming

The canonical pattern decouples the **synchronous REST call that establishes an observation** from the **asynchronous delivery of notifications**. The underlying device behaviour is CoAP Observe (`RFC 7641`).

Establish:

```
POST /v1/tenants/{t}/devices/{ep}/{o}/{i}/{r}:observe
POST /v1/tenants/{t}/devices/{ep}:observe-composite
  -> 201 Created
     Location: .../observations/{observationId}
```

- Notifications then arrive **out-of-band** over a push channel (see streaming below), **not** as the body of the establishing request.
- Cancel by the observation id (mapped to `DELETE` / Cancel-Observe).

This separation lets a single synchronous "start observing" call fan out into an open-ended asynchronous notification stream. See [architecture.md](architecture.md) for the end-to-end Observe/Notify flow on the device side.

---

## Event envelope and taxonomy

Notifications and lifecycle events are delivered as CloudEvents v1.0.2 envelopes.

| Attribute     | Status | Notes |
|---------------|--------|-------|
| `type`        | REQUIRED | Event type identifier |
| `source`      | REQUIRED | Context that emitted the event |
| `id`          | REQUIRED | Unique per `source` |
| `specversion` | REQUIRED | `"1.0"` |
| `time`        | OPTIONAL | A time-less envelope is spec-legal |
| `data`        | OPTIONAL | A data-less envelope is spec-legal |

A consumer **MUST NOT reject** a time-less or data-less envelope — both are valid per the spec.

Domain correlation is carried in **extension attributes**: tenant, device endpoint, observation id, and LwM2M version. A W3C Trace Context `traceparent` (format `version-traceId-spanId-flags`) is propagated for end-to-end tracing across the CoAP↔HTTP boundary.

```json
{
  "specversion": "1.0",
  "type": "com.example.lwm2m.notify.v1",
  "source": "/v1/tenants/acme/devices/dev-01",
  "id": "evt-91c0",
  "tenant": "acme",
  "endpoint": "dev-01",
  "observationId": "obs-7f3a",
  "lwm2mVersion": "1.1",
  "traceparent": "00-4bf92f3577b34da6a3ce929d0e0e4736-00f067aa0ba902b7-01",
  "data": { "/3/0/9": 87 }
}
```

### Event-taxonomy caution

A server may **collapse** all firmware/software lifecycle telemetry into a **single flat event type** (for example `...lwm2m.fota.v1`) with **no** per-stage `campaign-*` or `update.v1` types, and may **fold** registration updates into the registration event type. Consequently a dashboard must **derive** campaign progress by **aggregating per-device events** rather than relying on one event type per logical lifecycle stage. Do not assume a one-to-one mapping between logical lifecycle stages and emitted event types.

---

## SSE delivery mechanics for browser clients

For browser and operator clients, events are typically delivered over Server-Sent Events (SSE). The skill's architecture.md previously documented SSE only in passing; the mechanics below are the operational detail (**implementation pattern**).

### EventSource cannot authenticate

The browser-native `EventSource` API **cannot set request headers**, so a bearer/DPoP-authenticated stream cannot use it. Drive the stream instead with `fetch()` + a `ReadableStream` reader:

```
GET /v1/events
Accept: text/event-stream
Authorization: DPoP <token>
DPoP: <proof>
```

Client responsibilities:

- Own its **reconnect/backoff** (the manual `fetch` path has no built-in reconnect).
- Resend `Last-Event-ID` on reconnect to resume.
- Tolerate `:keepalive` comment frames, **multi-line** `data:` fields, and server `retry:` hints during frame parsing.
- **Reset backoff** on a successful (re)open.

### Topology must be deliberate

The SSE/event sink is frequently mounted on a **separate port** from the main NBI API, so a naive same-origin `GET /v1/events` assumption returns `404`. Resolve this explicitly:

- Either mount `/v1/events` on the NBI port, **or**
- Document a separate sink and have the reverse proxy route `/v1/events` → sink.

Proxy **match order is significant**: the `/v1/events` → sink rule **MUST** be ordered **before** the catch-all `/v1/*` → api rule, or the catch-all swallows the event path.

```
# reverse-proxy rule order (significant)
location = /v1/events { proxy_pass http://event-sink; }   # MUST come first
location   /v1/       { proxy_pass http://nbi-api;     }   # catch-all after
```

---

## Content negotiation for browsers

Composite-read responses default to `application/senml+cbor`. A browser has **no native CBOR decoder**, so a browser client **MUST** content-negotiate `application/senml+json` via a per-request `Accept` override.

SenML decoding rules a correct client must honour:

- Carry the base name `bn` **forward** across records.
- Concatenate `bn` + `n` to reconstruct the **absolute** `/o/i/r` resource path.
- **Preserve falsy values** — `v: 0`, `vs: ""`, and `vb: false` are **valid present values**, not missing fields. A decoder that drops falsy values corrupts the read.

```
Accept: application/senml+json     # browser override; default is senml+cbor
```

See [operation-coap-mapping.md](operation-coap-mapping.md) for the SenML content-format codes and the CoAP-side encoding.

---

## EST credential surface

A LwM2M credential NBI typically exposes Enrolment over Secure Transport (`RFC 7030`) alongside device credentials:

| Surface | Contents |
|---------|----------|
| Device credentials | PSK / RPK / X.509 |
| EST operations | enroll, re-enroll, cacerts |
| Server credentials | certificate + PSK-hint |

Caution: EST is, in practice, usually **enrolment-only**. Revocation surfaces — CRL / OCSP, and `RFC 7030` §4.2.3 — are **commonly missing**. Treat "EST present" as **enrol-only** until revocation has been independently verified.

---

## API strategy and migration

**Implementation guidance.** A maturing platform commonly carries **two NBI surfaces** in parallel:

| Surface | Character |
|---------|-----------|
| Legacy `/api` | Broad, ad-hoc, pre-standard |
| Standardised `/v1` | OMA NBI direction — OAuth 2.1 + DPoP, `RFC 9457` errors, tenant-scoped |

Recommendation: converge onto the standardised NBI and gate the migration behind **runtime feature flags** for gradual rollout — for example `useNbiApi`, `useCompositeOps`, `useCloudEvents`.

**Pin the response-envelope shape per endpoint.** Mismatched envelopes silently break consumers. Common shapes:

- a bare array
- a single-key wrapper
- `{ items, total, page, pageSize, hasMore }`

Align test mocks to the **real contract**, not the reverse — mocks that drift from the server contract hide breakage until production.

---

## Key specification references

| Reference | Use in the NBI |
|-----------|----------------|
| `RFC 9457` | Problem Details for HTTP APIs — error model (extended with `lwm2m:coapCode`) |
| `RFC 9449` | OAuth 2.0 Demonstrating Proof of Possession (DPoP) — `htu` §4.2, `ath` §4.3 |
| `RFC 7641` | Observing Resources in CoAP — device-side basis for Observe→streaming |
| `RFC 7638` | JSON Web Key (JWK) Thumbprint — DPoP `jkt` |
| `RFC 7030` | Enrolment over Secure Transport (EST) — credential surface; revocation §4.2.3 |
| `OMA-TS-LightweightM2M_Core` | LwM2M operation semantics (Read, Write, Execute, Create, Delete, Observe, Discover); version gating of Composite/Send |
| CloudEvents v1.0.2 | Event envelope for notification/lifecycle streaming |
| W3C Trace Context | `traceparent` propagation across the CoAP↔HTTP boundary |

`TODO(verify):` the exact section number(s) of the OMA LwM2M v2.0 Northbound API companion specification that normatively define the REST verb mapping and the tenant-scoped URI template — encoded here as **implementation pattern** pending that citation.

---

## Related files

- [architecture.md](architecture.md) — Northbound API integration patterns, Observe/Notify flows, server architecture
- [protocol-details.md](protocol-details.md) — transport security and the DTLS handshake path
- [security.md](security.md) — credential handling, DPoP/key-verification discipline
- [operation-coap-mapping.md](operation-coap-mapping.md) — CoAP operation/content-format mapping underlying the REST verbs
- [versions.md](versions.md) — per-version capability matrix (Composite/Send gating behind 412)
- [objects.md](objects.md) — Access Control Object (`/2`) device-side authorisation, distinct from operator RBAC
