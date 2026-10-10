# LwM2M Security — Comprehensive Reference

## Table of Contents
1. [Security Architecture Overview](#security-architecture-overview)
2. [Normative Security Requirements](#normative-security-requirements)
3. [Security Modes](#security-modes)
4. [DTLS 1.2 with Connection ID (RFC 9146)](#dtls-12-with-connection-id-rfc-9146)
5. [DTLS 1.3 (RFC 9147)](#dtls-13-rfc-9147)
6. [TLS/DTLS Alert Handling](#tlsdtls-alert-handling)
7. [TLS for TCP Bindings](#tls-for-tcp-bindings)
8. [OSCORE (RFC 8613)](#oscore-rfc-8613)
9. [Credential Provisioning](#credential-provisioning)
10. [Access Control](#access-control)
11. [DTLS Library Comparison](#dtls-library-comparison)
12. [Session Persistence & NVM](#session-persistence--nvm)
13. [Common Security Pitfalls](#common-security-pitfalls)
14. [Northbound/API Security](#northboundapi-security)

---

## Security Architecture Overview

LwM2M security operates at three layers:

1. **Transport security:** DTLS (UDP), TLS (TCP), or OSCORE (any transport)
2. **Credential management:** Security Object (/0) stores per-server credentials
3. **Authorization:** Access Control Object (/2) enforces per-object per-server permissions

All LwM2M communication between Client, Server and Bootstrap-Server MUST be mutually authenticated, encrypted and integrity-protected (Transport TS 1.2.2 §5.1). NoSec (Security Mode 3) is **not** a development-only mode in the specification: it is permitted only where equivalent security is provided outside LwM2M (for example at the link layer), and the deployment MUST then use those alternative mechanisms (Transport TS 1.2.2 §5.2.9.4). See [Mode 3](#mode-3-nosec).

---

## Normative Security Requirements

Transport TS 1.2.2 §5.1 states the requirements any LwM2M security solution must meet, independent of whether TLS/DTLS or OSCORE delivers them:

| Requirement | Statement (Transport TS 1.2.2 §5.1) |
|---|---|
| Mutual authentication | Client MUST authenticate Server and Bootstrap-Server; Server and Bootstrap-Server MUST authenticate Client — all **prior to exchange of any data** |
| Confidentiality + integrity | Each party MUST encrypt and integrity-protect data sent to the other, on both the Server and Bootstrap-Server paths |
| Replay protection | A security solution MUST be able to provide replay protection of LwM2M Operations |
| Request/response binding | A security solution MUST be able to securely bind LwM2M responses to LwM2M requests |
| Freshness | For certain operations the Client MUST be able to verify freshness of the request (see §5.6) |
| Secure fragmentation | A security solution MUST support fragmentation into separately verifiable fragments, in particular for firmware update |
| Intermediaries | The requirements also apply across LwM2M-aware and LwM2M-unaware intermediate nodes |
| Data at rest | Secrets and sensor data at rest must also be secured against unauthorised access |

Related normative statements elsewhere in Transport TS 1.2.2:

- **Distinct key pairs per entity (§5.2.1):** Bootstrap-Servers, Servers and Clients MUST use different key pairs; each Client MUST use keys unique to that Client. Using different credentials per Server is recommended for unlinkability. Implementations SHOULD conform to RFC 7925 (ciphersuites and extension recommendations).
- **Security-state retention (§5.2.1):** TLS/DTLS peers SHOULD keep session keys, sequence numbers and IVs for as long as can safely be achieved; if that state persists across a RAM-off sleep, secure storage SHOULD be used.
- **Blockwise identification (§5.5):** Blockwise transfers are vulnerable to block interchange between requests when the replay window exceeds 1 (applies to both DTLS and OSCORE; not to TCP). Clients and Servers supporting Blockwise SHOULD implement the CoAP Request-Tag option (RFC 9175).
- **Freshness (§5.6):** Client and Server MUST be able to verify freshness of certain operations. DTLS/OSCORE anti-replay rejects duplicates but still accepts unordered delivery within the window, so a maliciously delayed Write can be injected later. The Echo option (RFC 9175) is the LwM2M mechanism: MUST on first use of an OSCORE context, SHOULD where an operation needs freshness that cannot otherwise be guaranteed (§5.4.3).
- **Endpoint Client Name binding (§5.2.6):** `ep` is recommended but not mandated in Bootstrap-Request, Bootstrap-Pack-Request and Register. Because it is not authenticated at the application layer, the Server MUST compare the received `ep` with the identity used in the TLS/DTLS handshake (equality or a lookup table) and MUST answer **4.00 Bad Request** on mismatch. The same rule applies under OSCORE against the Sender ID (§5.4.5). In NoSec the comparison is against the network-access identity (§5.2.9.4).
- **Roles (§5.2.7):** the LwM2M Client is always the TLS/DTLS client; Server and Bootstrap-Server are always the TLS/DTLS server.
- **Curves (§5.2.3):** secp256r1 SHALL be supported for ECDHE and ECDSA; groups under 255 bits SHALL NOT be supported.

---

## Security Modes

Security Mode is Resource 2 of the Security Object (`/0/x/2`, Integer 0..4, OMA-SUP-XML_0 v1.2). The Security Object has **no device-management operations**: its resources MUST only be changed by a Bootstrap-Server or Bootstrap from Smartcard and MUST NOT be accessible by any other LwM2M Server (Core TS 1.2.2 Appendix E.1; the Operations column in OMA-SUP-XML_0 is blank for every resource). Table 5.2.4-1 of Transport TS 1.2.2 summarises which of R3/R4/R5 each mode populates:

| Mode | Public Key or Identity (R3, Opaque) | Secret Key (R5, Opaque) | Server Public Key (R4, Opaque) |
|---|---|---|---|
| 0 PSK | PSK identity | PSK | not used (ignored) |
| 1 RPK | client RPK | client private key | server RPK |
| 2 X.509 | client certificate (+ chain) | client private key | trust anchor or server certificate |
| 3 NoSec | not used | not used | not used |
| 4 EST | not used | not used | server certificate |

### Mode 0: Pre-Shared Key (PSK)
- Simplest to deploy, lowest overhead
- Symmetric key shared between client and server
- Security Object /0 resources: Resource 3 = PSK Identity (type **Opaque** per OMA-SUP-XML_0; RFC 7925 identity semantics), Resource 5 = PSK (Opaque). Resource 4 (Server Public Key) MUST NOT be used; a Bootstrap-Server is RECOMMENDED to omit it and the client MUST ignore any value written (Transport TS 1.2.2 §5.2.9.1)
- Identity and key sizes: Clients and Servers MUST support arbitrary PSK identities up to 128 bytes and PSK keys up to 64 bytes (RFC 7925 mandates). Provisioning a 16-byte (128-bit) or longer key is RECOMMENDED because the default ciphersuite uses AES-128 (§5.2.9.1)
- Ciphersuites: a Server supporting PSK MUST support TLS_PSK_WITH_AES_128_CCM_8 (RFC 6655) and TLS_PSK_WITH_AES_128_CBC_SHA256 (RFC 5487). A v1.1/v1.2 Client MUST support TLS_PSK_WITH_AES_128_CCM_8 and MAY support others. The Client SHOULD NOT use TLS_PSK_WITH_AES_128_CBC_SHA256 (RFC 7457 attacks) (§5.2.9.1)
- Bootstrap PSKs MUST be strong (high-entropy); the PSK ciphersuites MUST NOT be used with a low-entropy secret or a password (§5.2.4)
- Suitable for constrained devices with factory-provisioned keys
- Limitation: key rotation requires re-provisioning (bootstrap or out-of-band)

### Mode 1: Raw Public Key (RPK)
- Asymmetric keys without full certificate infrastructure
- Each side presents a raw public key (SubjectPublicKeyInfo structure)
- Security Object /0 (Transport TS 1.2.2 §5.2.9.2): Resource 3 = client RPK encoded as SubjectPublicKeyInfo (RFC 7250); Resource 5 = client private key encoded as OneAsymmetricKey (RFC 5958); Resource 4 = server RPK as SubjectPublicKeyInfo
- Lower overhead than x509 (no certificate chain validation)
- Ciphersuites: a Server supporting RPK MUST support TLS_ECDHE_ECDSA_WITH_AES_128_CCM_8 (RFC 6655/7925) and TLS_ECDHE_ECDSA_WITH_AES_128_CBC_SHA256 (RFC 5289); a v1.1/v1.2 Client MUST support TLS_ECDHE_ECDSA_WITH_AES_128_CCM_8. Ciphersuites SHOULD use ECDSA authentication and ECDHE key exchange (§5.2.9.2)
- Trust established by pre-provisioning the expected public key: the client MUST check the server's presented RPK matches Resource 4; the server MUST hold a stored copy of the expected client key and MUST check the presented RPK exactly matches it (§5.2.9.2)

#### RPK trust model (RFC 7250)

RPK does **not** send a certificate. The peer transmits a **bare `SubjectPublicKeyInfo`** structure — an RFC 5480 `SEQUENCE` of `AlgorithmIdentifier` + `BIT STRING` (the EC point) — and nothing else. It is **not** a self-signed X.509 certificate.

- RPK is negotiated via the `client_certificate_type` and `server_certificate_type` TLS extensions (RFC 7250 §3). Each side advertises `RawPublicKey` as an acceptable certificate type; if the peer does not, the handshake falls back to X.509 or fails.
- Because there is **no chain and no CA**, trust is established **out-of-band** (RFC 7250 §6): the received SPKI must byte-match (or fingerprint-match) a public key already held in a pre-provisioned credential store. A key that is absent from the store, or present but untrusted, **must reject the handshake**.
- A **self-signed X.509 certificate used as a trust overlay is NOT genuine RPK** — it is X.509 mode (Mode 2) with a self-issued anchor. Genuine RPK carries no ASN.1 certificate wrapper at all.
- Implementation note: stacks commonly accept a configured X.509 certificate purely as a container, then **extract the SPKI** (e.g. OpenSSL `X509_get_pubkey` followed by `i2d_PUBKEY`) and load **only those SPKI bytes** as the RPK. The certificate's signature, validity dates, and subject are discarded.

### Mode 2: x509 Certificate
- Full PKI with X.509v3 certificates (RFC 5280), SHOULD follow the RFC 7925 profile; secp256r1 SHALL be supported, groups under 255 bits SHALL NOT (Transport TS 1.2.2 §5.2.9.3)
- Security Object /0 (§5.2.9.3): Resource 3 = client end-entity certificate and, if available, some or all of the chain up to the root CA; Resource 5 = client private key as a **DER-encoded OneAsymmetricKey version 1** (RFC 5958 §2 — version 2 and the encrypted private-key structure of RFC 5958 §3 are not supported); Resource 4 = either a trust anchor for path validation of the server certificate, or the server certificate itself ("domain-issued certificate mode"). DER encoding MUST be supported by Client and Bootstrap-Server for R3 and R4; PEM (RFC 7468) MAY be supported and allows multiple certificates in one value
- Ciphersuites: a Server supporting X.509 MUST support TLS_ECDHE_ECDSA_WITH_AES_128_CCM_8 (RFC 7251/7925) and TLS_ECDHE_ECDSA_WITH_AES_128_CBC_SHA256 (RFC 5289); a v1.1/v1.2 Client MUST support TLS_ECDHE_ECDSA_WITH_AES_128_CCM_8; ECDSA authentication and ECDHE key exchange SHOULD be used (§5.2.9.3)
- Client certificates SHOULD carry the Endpoint Client Name in the subject CN; the Server MUST apply the §5.2.6 `ep` ↔ handshake-identity check (4.00 on mismatch)
- Server identity check: the TLS/DTLS client MUST implement the RFC 6125 / RFC 7925 §4.4.1 reference-identifier vs presented-identifier algorithm and path validation. The comparison is skipped only when the certificate in Resource 4 byte-matches the certificate presented in the handshake (domain-issued mode). If the server is provisioned by FQDN, the server certificate MUST contain that FQDN (SubjectAltName or CN) (§5.2.9.3)
- **Matching Type `/0/x/13`** (v1.1+, Unsigned Integer 0..3): how Resource 4 is presented — 0 exact value (default, v1.0 behaviour), 1 SHA-256 hash, 2 SHA-384 hash, 3 SHA-512 hash (OMA-SUP-XML_0 v1.2)
- **Certificate Usage `/0/x/15`** (v1.1+, 0..3, semantics reused from RFC 6698 per Transport TS 1.2.2 §5.2.9.6.4): 0 CA constraint, 1 service certificate constraint (PKIX validation + match), 2 trust anchor assertion, 3 domain-issued certificate (match only, no PKIX validation). **Absent ⇒ 3** (OMA-SUP-XML_0 v1.2)
- **SNI `/0/x/14`** (String, v1.1+): see [SNI](#sni-0x14). Required by Transport TS 1.2.2 only for deployments without DNS (§5.2.9.6.1)
- Certificate expiry needs a real-time clock and a time reference; see [Certificate expiry and revocation](#certificate-expiry-and-revocation)
- Highest overhead but best for large-scale managed deployments

#### SNI (/0/x/14)

- The resource exists since Security Object v1.1 (LwM2M 1.1). When present, the LwM2M Server URI is the service address and the SNI value is used to match the presented certificate or PSK identity (OMA-SUP-XML_0 v1.2 R14).
- Transport TS 1.1, 1.1.1 and 1.2 §5.2.9.3 stated "The SNI extension MUST be set by the LwM2M client" (verified in Transport TS 1.1/1.1.1/1.2 text). That blanket MUST was dropped in Transport TS 1.2.1 and is absent from 1.2.2.
- Transport TS 1.2.2 retains a conditional MUST: **for deployments without DNS** (client connects to an IP literal in the Server URI), the TLS/DTLS client and server MUST support the SNI extension (RFC 6066). The SNI value becomes the reference identifier; the client matches it against the identifier in the server certificate and aborts with a fatal alert on mismatch (§5.2.9.6.1).
- RFC 6066 §3 permits only an FQDN hostname in ServerName, so an IP literal cannot be used as the SNI value — hence the dedicated resource.

### Mode 3: NoSec
- No LwM2M-layer security: neither TLS/DTLS nor OSCORE is applied
- Transport TS 1.2.2 §5.2.9.4: "The LwM2M protocol MUST NOT be deployed without appropriate security." NoSec is permitted only because security may be provided **outside the scope of LwM2M** (e.g. at lower layers); in that case the deployment MUST use alternative security mechanisms
- The Server MUST compare the `ep` used in Register / Bootstrap-Request / Bootstrap-Pack-Request with the identifier used for network-access authentication (§5.2.9.4)
- Resource 2 MUST be 3; the Bootstrap-Server is RECOMMENDED to omit R3/R4/R5 and the client MUST ignore any values written
- Where it is used normatively: the **LoRaWAN binding uses NoSec** — security is provided by the LoRaWAN transport (Transport TS 1.2.2 §6, LoRaWAN binding). **SMS NoSec** is permitted for SMS triggering when all other traffic is secured over UDP/TCP, with a DoS caveat (§5.3, §5.3.1); SMS-only clients MAY use NoSec for debugging but otherwise MUST support SMS Secured Mode (§5.3). OSCORE over an otherwise NoSec DTLS layer is the OSCORE-only configuration (§5.4.1)
- For plain CoAP over UDP/TCP on an open network with no lower-layer protection, NoSec violates §5.1 and §5.2.9.4 — unacceptable in production

### Mode 4: Certificate mode with EST (v1.1+)
- Introduced with Security Object v1.1 in LwM2M 1.1 (verified: Transport TS 1.1 §5.2.4 lists "4: Certificate mode with EST"); EST over CoAP is RFC 9148 ([CoAP-EST] in the Transport TS 1.2.2 reference list)
- Resource 2 MUST be 4 and the **TLS/DTLS server certificate MUST be provisioned to Resource 4 (Server Public Key)**. This triggers the client to generate a public/private key pair locally and run an EST-coaps exchange to obtain its own certificate; once obtained, the mode behaves like Mode 2 (Transport TS 1.2.2 §5.2.9.5)
- **Resources 3 and 5 are not used** in Mode 4; the Bootstrap-Server is RECOMMENDED to omit them and the client MUST ignore any value written (§5.2.9.5)
- The private key SHOULD never leave the device: the client SHOULD use an existing key pair or generate one locally (§5.2.9.5)
- Over-the-air cost: SimpleEnroll / SimpleReEnroll carry the public key plus a PKCS#10 proof-of-possession, which is heavier than Mode 2 but keeps the private key on-device (§5.2.9.5)
- EST features: **Simple PKI messages and CA certificate retrieval SHALL be supported** when EST-coaps is used for bootstrapping. Server-generated key requests are not needed (covered by Modes 1/2) and CSR Attributes Request is not required because the Bootstrap-Server already holds the attributes (§5.2.9.5)
- The Transport TS does not describe which credential the client authenticates to the EST server with before enrolment; the §5.2.9.5 text covers only the server certificate in Resource 4 (unverified against OMA spec library)

---

## DTLS 1.2 with Connection ID (RFC 9146)

### Purpose
CID solves the NAT rebinding problem for sleepy devices. When a device enters PSM sleep and later wakes with a new source IP/port, the DTLS session would normally be lost (5-tuple mismatch). CID allows the session to survive address changes.

### What LwM2M actually says about CID
Transport TS 1.2.2 §5.2.8 is **descriptive, not normative**: it explains that DTLS 1.2 identifies the association by IP/port, that NAT rebinding after extended sleep forces a new handshake, and that [DTLS-1.2-CID] (RFC 9146) and [DTLS-1.3] (RFC 9147) define a Connection ID carried in the record-layer header so a client can keep an established security context longer. There is **no LwM2M MUST/SHOULD on CID**; everything below is RFC 9146 behaviour plus deployment advice. CID was introduced into the Transport TS with LwM2M 1.2.

### CID is a DTLS 1.2 extension
RFC 9146 Connection ID is defined as a **DTLS 1.2** extension. An mbedTLS-era stack may therefore need to be **pinned to DTLS 1.2** to use CID at all (DTLS 1.3 integrates the connection identifier natively — see [DTLS 1.3](#dtls-13-rfc-9147)). CID lets a session survive **NAT rebinding / IP-port migration** for a sleepy or roaming device **without a re-handshake**, which is essential for NB-IoT/cellular **PSM and eDRX wake** cycles where the device frequently reappears on a new source port.

### Negotiation Flow
```
Client                                          Server
  │                                                │
  │── ClientHello ──────────────────────────────►  │
  │   + connection_id extension (CID_C, may be     │
  │     zero-length — client need not receive one) │
  │                                                │
  │◄── HelloVerifyRequest + cookie ──────────────  │
  │                                                │
  │── ClientHello (with cookie) ────────────────►  │
  │   + connection_id extension (CID_C)            │
  │                                                │
  │◄── ServerHello ────────────────────────────── │
  │    + connection_id extension (CID_S)           │
  │    + Certificate / ServerKeyExchange           │
  │    + ServerHelloDone                           │
  │                                                │
  │── ClientKeyExchange / ChangeCipherSpec ─────►  │
  │── Finished ─────────────────────────────────►  │
  │                                                │
  │◄── ChangeCipherSpec ─────────────────────────  │
  │◄── Finished ─────────────────────────────────  │
  │                                                │
  │   [After handshake: records carry CID]         │
  │   ContentType = tls12_cid (25)                 │
  │   Record header includes CID                   │
  │                                                │
  │   ... device enters PSM sleep ...              │
  │   ... NAT rebinding occurs ...                 │
  │   ... new source port on wake ...              │
  │                                                │
  │── Application Data (CID=CID_S) ────────────►  │
  │   Server matches CID_S → existing session      │
  │   No handshake required                        │
```

### CID Rules (RFC 9146 behaviour — RFC text not in the OMA spec library, verify against RFC 9146)
- Each side's `connection_id` extension carries the CID **it wants to receive**. The value may be zero-length, which means "I support CID but do not need to receive one"
- **A zero-length client CID is legitimate and is the normal choice for a LwM2M Client.** What enables NAT-rebind recovery is the **server-issued CID (CID_S)** that the client places in every record it sends: the server looks up the security association by CID_S instead of by 5-tuple. The client itself does not need a CID of its own, because the server's address does not change
- If the server omits the extension → CID not negotiated, 5-tuple binding only
- CID-bearing records use ContentType 25 (`tls12_cid`) instead of 23 (`application_data`)
- CID is placed in the record header, before the encrypted payload
- CID length is chosen by the receiving side during the handshake and fixed for the session
- LwM2M adds no requirement on top of this (Transport TS 1.2.2 §5.2.8); whether to request CID is an implementation/deployment decision

### TLS Extension Type Values: 53 vs 54

**This is a common source of confusion.** The IANA TLS ExtensionType registry has two entries related to CID:

- **Extension Type 53** — Was allocated via early allocation for an **earlier draft** version of the CID specification (draft-ietf-tls-dtls-connection-id). This draft used a different wire format that is **incompatible** with the final RFC 9146. Some older implementations (notably earlier TinyDTLS builds and some mbedTLS versions) may still use extension type 53.
- **Extension Type 54 (`connection_id`)** — The **final, standards-track allocation** per RFC 9146. This is the value that MUST be used for conformant implementations. The extension is marked DTLS-Only=Y and Recommended=N (because it targets specific use cases, not general TLS).

**Interop implications:** If a client sends extension type 53 and the server expects 54 (or vice versa), the CID negotiation silently fails — the server simply does not echo the extension in ServerHello, and the session falls back to 5-tuple binding. This is a known cause of CID failures in mixed-version deployments.

**ContentType 25 (`tls12_cid`):** Registered by RFC 9146 in the TLS ContentType registry. When CID is active, all encrypted records use ContentType 25 instead of 23 (`application_data`). The CID field immediately follows the record header, before the encrypted payload. The receiver uses the CID value to look up the security association, bypassing the 5-tuple entirely.

### Return Routability Check (RFC 9853)

> **IETF, not referenced by any OMA LwM2M TS.** RFC 9853 does not appear in the Transport TS 1.2.2 reference list; the following is IETF guidance and vendor practice, not an OMA requirement (unverified against OMA spec library).

RFC 9853 defines a Return Routability Check (RRC) subprotocol for DTLS 1.2 and 1.3 that complements CID. When a CID-bearing record arrives from a new source address (after NAT rebinding), the server should verify the new peer address is legitimate before sending large responses. Without RRC, an attacker could spoof the source address to use the server as an amplification vector.

**RRC interaction with LwM2M:**
- Implementations offering CID SHOULD also negotiate the `rrc` extension (per RFC 9853)
- An alternative to RRC is CoAP Echo (RFC 9175), which provides application-layer address validation
- LwM2M servers with CID support should implement one of these mechanisms to prevent amplification attacks
- The RRC subprotocol uses `path_challenge` and `path_response` messages, inspired by QUIC (RFC 9000)

### Server-Side CID Routing
For clustered server deployments, CID can encode a node-ID prefix. When a CID-bearing record arrives, the load balancer can route it to the correct server node based on the CID prefix. Reference: Eclipse Californium's CID cluster routing implementation.

---

## DTLS 1.3 (RFC 9147)

Optional support added in LwM2M v1.2.

Transport TS 1.2.2 §5.2.1: "This version of the specification MAY optionally use TLS 1.3 [RFC 8446] and/or DTLS 1.3 [DTLS-1.3]", where [DTLS-1.3] = RFC 9147. The Transport TS gives no further DTLS 1.3 profile; the behaviour below is RFC 9147, whose text is not in the OMA spec library (verify against RFC 9147).

### Key Differences from DTLS 1.2 (per RFC 9147, verify)
- Reduced handshake round-trips (1-RTT, 0-RTT resumption)
- No ChangeCipherSpec messages
- Handshake messages encrypted earlier
- CID is part of the base protocol: the **same `connection_id` extension** (type 54) is negotiated, and CID-bearing records use the DTLS 1.3 **unified header** rather than ContentType 25; RFC 9147 also defines **CID update messages** (`RequestConnectionId` / `NewConnectionId`) so CIDs can be changed post-handshake
- Improved key schedule using HKDF
- Forward secrecy mandatory (no static RSA key exchange)

### LwM2M Considerations
- DTLS 1.3 CID is carried differently on the wire from DTLS 1.2 CID (RFC 9146) even though the negotiation extension is the same — a 1.2-only CID implementation does not interoperate with a 1.3 record layer
- Deployment advice (not an OMA requirement): a server fleet migrating to DTLS 1.3 should continue to accept DTLS 1.2 clients, since the Transport TS only makes 1.3 optional
- Security Object controls (OMA-SUP-XML_0 v1.2; all bootstrap-only):
  - `/0/x/16` DTLS/TLS Ciphersuite (multiple, Unsigned Integer): IANA ciphersuite codes the client should propose in ClientHello (e.g. TLS_PSK_WITH_AES_128_CCM_8 = 0xC0,0xA8 → 0xC0A8 = 49320)
  - `/0/x/18` Groups To Use by Client, `/0/x/19` Signature Algorithms Supported by Server, `/0/x/20` Signature Algorithms To Use by Client, `/0/x/21` Signature Algorithm Certs Supported by Server (all multiple, 0..65535)
  - `/0/x/22` **TLS 1.3 Features To Use by Client** (bitmask): bit 0 PSK Plain, bit 1 0-RTT, bit 2 PSK with PFS, bit 3 Certificate-based Authentication; bits 4–31 reserved. A `0` bit means the feature should not be used
  - `/0/x/23` TLS Extensions Supported by Server, `/0/x/24` TLS Extensions To Use by Client (bitmask of extension types)
  - `/0/x/14` is **SNI**, not a version constraint — there is no Security Object resource that pins the TLS/DTLS protocol version
- Not all DTLS libraries support 1.3 yet (see library comparison)

---

## TLS/DTLS Alert Handling

Transport TS 1.2.2 §5.2.10 gives default recommended handling for TLS/DTLS alerts (Table 5.2.10-1). **Fail** means the Client MUST consider the handshake failed and cannot recover by itself; **Retry** means the Client MUST consider the handshake failed but the condition MAY be transient and a retry may succeed. In both cases the Client MUST follow the error-handling procedures configured in the Server Object (e.g. Bootstrap on Registration Failure `/1/x/16`); no such fallback exists for the Client ↔ Bootstrap-Server path. Implementations may cap retries and fall back to bootstrapping past a threshold, and should use exponential back-off.

| Alert | Name | Recommendation |
|---|---|---|
| 0 | close_notify | Retry |
| 10 | unexpected_message | Retry |
| 20 | bad_record_mac | Retry |
| 21 | decryption_failed | Retry |
| 22 | record_overflow | Retry |
| 30 | decompression_failure | Ignore (should never occur) |
| 40 | handshake_failure | Retry |
| 41 | no_certificate_RESERVED | Ignore (should never occur) |
| 42 | bad_certificate | **Fail** |
| 43 | unsupported_certificate | **Fail** |
| 44 | certificate_revoked | **Fail** |
| 45 | certificate_expired | **Fail** |
| 46 | certificate_unknown | **Fail** |
| 47 | illegal_parameter | Retry |
| 48 | unknown_ca | **Fail** |
| 49 | access_denied | **Fail** |
| 50 | decode_error | Retry |
| 51 | decrypt_error | Retry |
| 60 | export_restriction_RESERVED | Ignore (should never occur) |
| 70 | protocol_version | Retry |
| 71 | insufficient_security | Retry |
| 86 | internal_error | Retry |
| 90 | user_canceled | Retry |
| 100 | no_renegotiation | Retry (should not occur — renegotiation unused) |
| 110 | unsupported_extension | Ignore |
| 111 | certificate_unobtainable (RFC 6066) | Retry (only with the relevant extension) |
| 112 | unrecognized_name (RFC 6066) | **Fail** (only with the relevant extension) |
| 113 | bad_certificate_status_response (RFC 6066) | **Fail** (only with the relevant extension) |
| 114 | bad_certificate_hash_value (RFC 6066) | **Fail** (only with the relevant extension) |
| 115 | unknown_psk_identity (RFC 4279) | **Fail** |

Per RFC 6347, DTLS implementations SHOULD silently discard records with bad MACs or that are otherwise invalid (noted in §5.2.10).

---

## TLS for TCP Bindings

For CoAP over TCP (RFC 8323), TLS is used instead of DTLS:

- TLS 1.2 (RFC 5246) or, optionally, TLS 1.3 (RFC 8446) (Transport TS 1.2.2 §5.2.1)
- Same credential modes (PSK, RPK, x509, EST)
- SNI: same rule as DTLS — MUST be supported only for deployments without DNS (Transport TS 1.2.2 §5.2.9.6.1); see [SNI](#sni-0x14)
- No CID concept needed — TCP maintains the connection
- Keepalive via CoAP Ping/Pong over TCP

---

## OSCORE (RFC 8613)

Application-layer security for CoAP, independent of DTLS/TLS.

### Key Properties
- Protects CoAP at the message level (not the transport)
- Survives proxies, caches, and transport changes
- Uses COSE (CBOR Object Signing and Encryption) for encryption
- Works with any transport binding (UDP, TCP, SMS, Non-IP)
- Particularly useful for Non-IP transports where DTLS handshake is expensive
- Can be used alongside DTLS/TLS for defence-in-depth

### OSCORE Security Context
Stored in Object 21 (LwM2M OSCORE Object). Introduced in **LwM2M 1.1** (Transport TS 1.1 §5.5; Core TS 1.1 lists Object 21); current definition is **OMA-SUP-XML_21 v2.0** (2021-11-23, LWM2MVersion 1.1), which added Resource 6 and made the identifier resources Opaque. Like the Security Object it is bootstrap-only (no DM operations; MUST only be changed by a Bootstrap-Server or Bootstrap from Smartcard) and is **excluded from the Register "Objects and Object Instances" list** together with /0 and /23 (Core TS 1.2.2 Table 6.2.1-1). An Object 21 instance is linked from exactly one Security Object instance via `/0/x/17` OSCORE Security Mode (Objlnk) and MUST NOT be linked from more than one.

| Resource | Name | Type / Mand. | Purpose (OMA-SUP-XML_21 v2.0; Transport TS 1.2.2 §5.4.7.1) |
|----------|------|--------------|---------|
| 0 | OSCORE Master Secret | Opaque, M | Pre-shared Master Secret; same on both ends; MUST be populated when /0/x/17 is present |
| 1 | OSCORE Sender ID | Opaque, M | Client's Sender ID (Transport TS §5.4.7.1 describes it as a UTF-8 string value) |
| 2 | OSCORE Recipient ID | Opaque, M | Client's Recipient ID (= the Server's Sender ID, §5.4.6) |
| 3 | OSCORE AEAD Algorithm | Integer, O | COSE AEAD algorithm code (RFC 8152 Table 10); default AES-CCM-16-64-128, mandatory-to-implement (§5.4.2) |
| 4 | OSCORE HMAC Algorithm | Integer, O | COSE HMAC code (RFC 8152 Table 7) for the HKDF; HMAC-SHA-256 is the default (§5.4.2) |
| 5 | OSCORE Master Salt | Opaque, O | Non-secret random salt for context derivation |
| 6 | OSCORE ID Context | Opaque, O | ID Context identifying the Common Context (new in v2.0) |

### LwM2M-specific OSCORE rules (Transport TS 1.2.2 §5.4)
- **Optional** overall; MAY be used with a Bootstrap-Server, with a Server, and between a LwM2M endpoint and a non-LwM2M endpoint (bootstrapping of the latter out of scope) (§5.4.1)
- **Layering (§5.4.1):** with Security Mode 0–2 the message is protected by **both OSCORE and DTLS**; with Security Mode 3 (NoSec) it is protected by **OSCORE only**. OSCORE is independent of DTLS/TLS and survives proxies and transport translation (UDP, TCP, SMS, Non-IP, 3GPP CIoT)
- **`/0/x/17` link MUST (§5.4.7.1):** if the OSCORE Security Mode resource is present it MUST link to an Object 21 instance, and OSCORE MUST then be used with that Server (OMA-SUP-XML_0 R17). Its presence requires R0–R2 of Object 21 to be populated; R3–R5 MAY be populated
- **Bootstrapping (§5.4.3):** a Client supporting client- or server-initiated bootstrap MUST support a strong (high-entropy) pre-shared Master Secret, unique per device and per security protocol; a Bootstrap-Server MUST support PSK bootstrapping. Reuse conditions of RFC 8613 §3.3 MUST be complied with, and the Appendix B.2 procedure SHALL be followed when a context is used for the first time
- **Echo on first use (§5.4.3):** the receiving endpoint MUST use the Echo option when the OSCORE context is used for the first time (e.g. Bootstrap-Request, Bootstrap-Pack-Request, Register) and SHOULD use it when an operation (e.g. Write) needs freshness that cannot otherwise be guaranteed
- **Endpoint Client Name (§5.4.5):** the `ep` MAY be authenticated at the application layer by setting the Sender ID to the Endpoint Client Name. If it is not, the Server MUST compare the received `ep` with the Client's Sender ID (equality or lookup table) and MUST respond **4.00 Bad Request** on mismatch
- **Unbootstrapping (§5.4.4):** the §5.2.5 procedure applies unchanged
- **Credential model (§5.4.7):** from v1.1 the Master Secret is assumed pre-shared and is therefore known to the provisioning party (Bootstrap-Server); the trust model must accept that

### OSCORE vs DTLS
| Aspect | DTLS | OSCORE |
|--------|------|--------|
| Layer | Transport | Application |
| Handshake | Required (2+ RTT) | None (pre-shared context) |
| Proxy traversal | Terminates at proxy | End-to-end through proxies |
| Session state | 5-tuple or CID | Context IDs |
| Replay protection | Sequence numbers | Sequence numbers |
| Forward secrecy | With ECDHE | Not built-in |
| Best for | Direct client-server | Constrained/proxied networks |

---

## Credential Provisioning

### Bootstrap Methods

**Factory Bootstrap:** Credentials pre-provisioned during manufacturing. Most secure but least flexible. Requires secure manufacturing environment.

**Smartcard Bootstrap:** Credentials stored on SIM/UICC. Uses PKCS#15 structure. See OMA-TS-LightweightM2M_Core Appendix G.

**Client-Initiated Bootstrap:** Client connects to Bootstrap-Server using bootstrap credentials. BS provisions operational credentials. Most common for field-provisioned devices.

**Server-Initiated Bootstrap:** Server triggers bootstrap re-provisioning. Useful for credential rotation.

### Write-only credential resources

Credential resources in management-accessible objects should be exposed **Write-only (W)** so a server cannot read the secret back over the management interface. Verified examples: WLAN Connectivity `/12/x/17` WPA Pre Shared Key, `/12/x/18` WPA Key Phrase, `/12/x/21`–`/12/x/25` WEP keys and `/12/x/28` RADIUS Secret are all W (OMA-SUP-XML_12 v1.1.1); Software Management `/9/x/14` User Name and `/9/x/15` Password are W (OMA-SUP-XML_9 v1.0.3). Note the counter-example: APN Connection Profile `/11/x/6` Secret is **RW** in OMA-SUP-XML_11 v1.1 — a defect to be aware of when exposing that object. The Security Object `/0` and OSCORE Object `/21` are a different case: they carry **no DM operations at all** (blank Operations column) and MUST NOT be accessible by any LwM2M Server other than via Bootstrap (Core TS 1.2.2 Appendix E.1), so the PSK `/0/x/5` is never readable or writable over the management interface. The trust direction is one-way: secrets are pushed in, never pulled out.

On **embedded Linux**, never store secret bytes in world-readable configuration (e.g. OpenWrt UCI / `/etc/config`, which is typically world-readable). Instead store only **file paths** that point at **root-owned `0600`** files and read the bytes at runtime — **"path-not-payload"**. This keeps the secret material out of config backups, logs, and any process that can read the config tree.

### Credential change and unbootstrapping (Transport TS 1.2.2 §5.2.4, §5.2.5)

- **Credential change policy (§5.2.4):** dynamically provisioned credentials MAY change at any time, even during a live TLS/DTLS session. TLS/DTLS verifies credentials only at handshake (absent renegotiation), so an already-established record layer keeps working after a PSK or certificate changes. Whether to close the connection on credential change is an explicit **policy decision** for client and server implementations; the TS does not mandate either behaviour.
- **Bootstrap-Server obligations (§5.2.4):** a Client supporting client-initiated bootstrap MUST support at least one of (a) strong pre-shared secret, (b) RPK or certificate; a Bootstrap-Server MUST support all three credential types for full interoperability; Clients MUST be provisioned with device-unique credentials. The Security Object also protects the firmware repository connection when `/5/0/1` Package URI is used.
- **Unbootstrapping (§5.2.5)** — on Bootstrap-Delete of a Security Object instance the Client MUST: (1) delete the referenced Server Object instance (it MAY De-register first); (2) delete every Object Instance, and its Access Control instance, that only that Server could access (Server is ACO and sole ACL holder); (3) for shared Object Instances, delete that Server's ACL Resource Instance and, if it was the Access Control Owner, hand ownership to the Server with the highest sum of Write(1)+Delete(1) rights (tie → Client picks one); (4) delete that Server's observations. Servers MAY observe `/2/x/3` to track ownership changes. The same procedure applies to OSCORE (§5.4.4).

### Certificate expiry and revocation

Source: Transport TS 1.2.2 §5.2.9.6.2–§5.2.9.6.3.

- **Expiry (§5.2.9.6.2):** certificate use needs a real-time clock and a provisioned time reference. The Bootstrap-Server or an authorised Server MAY write the Device Object **Current Time** resource (`/3/0/13`) to provide that reference; the Client SHOULD verify the freshness of that Write. Setting Current Time is security-critical, so appropriate ACL settings are needed. There is **no in-band recovery from an expired Bootstrap-Server certificate** — out-of-band (firmware update, commissioning tool) only.
- **Revocation (§5.2.9.6.3):** three candidate approaches, none mandatory: (1) the Client MAY rely on the TLS/DTLS error-recovery path when server-certificate verification fails during the handshake; (2) the Client MAY use **OCSP stapling** (RFC 6961), which presupposes regular re-handshakes and adds handshake bytes; (3) the operator uses **server-initiated bootstrap** to force re-bootstrapping for new credentials (not usable by the Server whose own certificate was revoked). A revoked Bootstrap-Server certificate is again recoverable only out-of-band. The TS defines no CRL mechanism.

### EST over CoAP (Security Mode 4, v1.1+)
Per Transport TS 1.2.2 §5.2.9.5 and RFC 9148 (EST-coaps):
1. Bootstrap-Server writes `/0/x/2` = 4 and the TLS/DTLS server certificate into `/0/x/4` (Server Public Key); `/0/x/3` and `/0/x/5` are not used
2. Client generates (or reuses) a key pair locally — the private key SHOULD never leave the device
3. Client runs EST-coaps: CA certificate retrieval (`/crts`) and Simple PKI enrolment (`/sen`, PKCS#10 CSR with proof-of-possession); SimpleReEnroll (`/sren`) for renewal
4. EST server returns the client certificate
5. From then on the account behaves as Mode 2 (X.509), with the server certificate in Resource 4 acting as the matching target
6. The TS does not specify which credential the client presents to the EST server before enrolment (unverified against OMA spec library)

#### EST scope — what the TS requires

Transport TS 1.2.2 §5.2.9.5 lists four EST features — Simple PKI messages, CA certificate retrieval, CSR Attributes Request, server-generated key request — and makes only **Simple PKI messages and CA certificate retrieval SHALL** when EST-coaps is used for bootstrapping. Server-generated keys are not needed (Modes 1/2 already provision keys) and CSR Attributes Request is not required (the Bootstrap-Server already holds the attributes). **EST is an enrolment protocol; certificate revocation is not an EST function in LwM2M** — revocation handling is covered separately in §5.2.9.6.3 (OCSP stapling MAY, server-initiated bootstrap) as above. Operationally, treat "EST present" as enrol/re-enrol/CA-certs only and verify revocation handling through the §5.2.9.6.3 paths.

---

## Access Control

Object 2 (Access Control) provides per-Object-Instance authorization:

- Each Access Control Object Instance specifies permissions for one Object Instance (`/2/x/0` Object ID, `/2/x/1` Object Instance ID)
- Resource 2 (ACL) is multi-instance: one Resource Instance per server, keyed by Short Server ID; Resource Instance ID **0** holds the **default** access rights for servers that have no ACL Resource Instance of their own (OMA-SUP-XML_2 v1.1 R2; Core TS 1.2.2 §8.1.2.2)
- Permission bits (OMA-SUP-XML_2 v1.1 R2): 1st LSB **Read = 1** (also covers Observe and Write-Attributes), 2nd LSB **Write = 2**, 3rd LSB **Execute = 4**, 4th LSB **Delete = 8**, 5th LSB **Create = 16**
- The Access Control Owner (Resource 3, Short Server ID; 65535 = managed by Bootstrap) gets **full access rights only when it has no ACL Resource Instance of its own**. If an ACL Resource Instance exists for the owner, the owner gets exactly the rights in that instance (Core TS 1.2.2 §8.1.2 authorisation sequence, steps 1–2)
- Non-owner servers get their own ACL Resource Instance if present, otherwise the default instance (ID 0) if present, otherwise no rights (Core TS 1.2.2 §8.1.2 step 3)
- Create is checked against the ACL instance associated with the **Object** (Object Instance ID 65535, owner 65535 — bootstrap-provisioned); it is the only operation checked at object rather than instance level (Core TS 1.2.2 §8.1.2.1)
- In Access-Control-Disabled context (single server, no Object 2) the server gets full rights (Core TS 1.2.2 §8.1.2)

### Multi-Server Scenarios
When a client connects to multiple servers, Access Control is critical:
- Bootstrap-Server provisions Access Control instances
- Each server gets specific permissions per object instance
- A Read-only monitoring server might get only Read (1) permission
- A management server might get Read+Write+Execute (7)

---

## DTLS Library Comparison

> **Vendor status as of 2026-10, unverified.** Library capability, version and footprint figures below come from vendor documentation and field experience, not from any OMA specification. Re-check against each project's current release notes before relying on them.

| Feature | mbedTLS 3.x | TinyDTLS | wolfSSL 5.x | OpenSSL 3.x |
|---------|-------------|----------|-------------|-------------|
| DTLS 1.2 CID (RFC 9146) | ✅ (3.2+) | ⚠ (draft impl) | ✅ (5.4+) | ❌ |
| DTLS 1.3 | ❌ | ❌ | ⚠ (partial) | ⚠ (experimental, 3.2+) |
| PSK | ✅ | ✅ | ✅ | ✅ |
| RPK | ✅ | ✅ | ✅ | ❌ native |
| x509 | ✅ | ❌ | ✅ | ✅ |
| Session resumption | ✅ | Limited | ✅ | ✅ |
| RAM (PSK only) | ~40KB | ~10KB | ~50KB | ~200KB+ |
| Flash (PSK only) | ~60KB | ~20KB | ~80KB | ~500KB+ |
| MCU suitable | ✅ | ✅ | ✅ | ❌ |
| Linux suitable | ✅ | ✅ | ✅ | ✅ |

### Known Issues
**TinyDTLS:** No x509 support. CID follows draft, not final RFC 9146 — interop failures with servers expecting the final spec. Slow maintenance cadence. Memory leaks reported in Wakaama integration.

**mbedTLS 3.x:** CID stable from 3.2+ (`MBEDTLS_SSL_DTLS_CONNECTION_ID`). API breaking changes from 2.x→3.x. Handshake failures (-26880, -26624) on some embedded platforms. No DTLS 1.3.

**wolfSSL 5.x:** CID from 5.4.0 (`--enable-dtls-cid`). Partial DTLS 1.3. Dual-licensed (GPLv2 + commercial).

**OpenSSL 3.x:** No CID at all — unsuitable for sleepy devices. Too large for MCU targets. DTLS 1.3 experimental in 3.2+ (table marks it ⚠ accordingly, not production-ready).

### RPK (RFC 7250) provider support matrix

RFC 7250 Raw Public Key support varies sharply by stack — the generic "RPK" column above hides important gating details:

| Stack | RPK status | Gate / mechanism |
|-------|-----------|------------------|
| mbedTLS | ❌ Not offered | Reserves the extension numbers (19/20) only; **no negotiation and no key API** to actually drive RPK |
| OpenSSL | ✅ from **≥ 3.2** | `certificate_type` extension support added in 3.2 |
| wolfSSL | ✅ build-gated | Requires the **`HAVE_RPK`** build option |
| tinyDTLS | ✅ build-gated | Raw ECDSA key handlers gated behind a build flag; the exact flag name (e.g. `LWM2M_ENABLE_RPK`, default OFF) is an integration-layer convention — unverified against the tinyDTLS source |

**Key gotcha — "cipher present ≠ RPK negotiated":** a device can compile in the `TLS_ECDHE_ECDSA_WITH_AES_128_CCM_8` cipher that RPK depends on yet **still not offer RPK** on the handshake, because the DTLS engine gates RPK behind **its own build flag** which must be set **before the DTLS dependency is configured**. Presence of the ECDHE-ECDSA cipher tells you nothing about whether the `certificate_type` extensions are actually advertised. Verify by inspecting the ClientHello/ServerHello certificate-type negotiation, not the cipher list.

---

## Session Persistence & NVM

For sleepy devices using Queue Mode + CID, DTLS session material must survive sleep cycles:

### What to Persist (NVM)
- **Permanent:** PSK identity/key, certificates, private keys, Endpoint Client Name, SSID, Security Object Resources 0–30 (OMA-SUP-XML_0 v1.2) and OSCORE Object Resources 0–6 (OMA-SUP-XML_21 v2.0)
- **Session-scoped (persist for CID):** DTLS Session ID, Connection ID (both client and server CIDs), epoch, sequence numbers, master secret, cipher suite, compression method
- **Ephemeral (RAM only):** CoAP Message IDs, Tokens, Registration Location path, dedup cache, 5-tuple

### NVM Write Endurance
Flash memory has limited write cycles (typically 10K–100K for NOR, 1K–10K for NAND). Strategies:
- Write session material only before entering PSM sleep, not on every message
- Use wear-leveling filesystem (LittleFS, SPIFFS)
- Minimize persisted data size
- Consider FRAM/MRAM for frequently-written session data on MCU targets

---

## Common Security Pitfalls

1. **Misreading the zero-length client CID:** a zero-length `connection_id` from the client is legitimate — it means the client does not need to receive a CID. NAT-rebind recovery depends on the **server** issuing a CID that the client then carries in its records. The real pitfall is a server that omits the extension (CID not negotiated) or a client stack that drops the server CID after sleep. (RFC 9146 behaviour, not an OMA requirement — verify against RFC 9146.)

2. **NoSec without lower-layer security:** Security Mode 3 is only permitted where security is provided outside LwM2M (e.g. LoRaWAN link security) and the deployment MUST then use those alternative mechanisms (Transport TS 1.2.2 §5.2.9.4). Plain CoAP on an open network provides no authentication or encryption and breaches §5.1.

3. **PSK key reuse:** Using the same PSK across multiple devices. If one device is compromised, all devices are compromised. Transport TS 1.2.2 §5.2.1/§5.2.4 require per-client unique keys.

4. **No session persistence:** Not persisting DTLS session state to secure storage before sleep (Transport TS 1.2.2 §5.2.1 SHOULD). Every wake cycle then requires a full handshake — about 3 round-trips including the HelloVerifyRequest cookie exchange for DTLS 1.2 (RFC 6347, unverified in corpus) and several KB of data for certificate modes.

5. **Missing SNI:** for deployments where the Server URI is an IP literal (no DNS), SNI support is a MUST and the resource `/0/x/14` supplies the reference identifier (Transport TS 1.2.2 §5.2.9.6.1). Without it, certificate selection and identity matching on multi-tenant servers fail. The blanket "client MUST set SNI" of Transport TS 1.1–1.2 no longer applies.

6. **Missing Extended Master Secret:** RFC 7627 hardens DTLS 1.2 against triple-handshake attacks. This is **implementation good practice, not an OMA requirement** — RFC 7627 is not referenced by any LwM2M TS (verified absent from Core and Transport TS 1.2.2 references).

7. **Ignoring CID in cluster routing:** In multi-node server deployments, CID-bearing packets must be routed to the node holding the session. Without CID-aware routing, sessions break after NAT rebinding.

8. **Certificate chain validation skipped:** Some constrained implementations skip certificate path validation for performance. This defeats the purpose of x509 security.

9. **OSCORE context not synchronised:** If client and server OSCORE sequence numbers diverge (e.g., after a crash), replay protection blocks legitimate messages. Implement sequence number recovery per RFC 8613 Appendix B.2.

10. **Bootstrap credentials as operational credentials:** The bootstrap PSK/cert should be different from operational credentials. Using the same credentials for both roles weakens the security boundary.

11. **"Configured ≠ functional" DTLS:** A server can have a PSK seeded and Security Objects (/0) populated yet **never actually perform a DTLS handshake** — the credentials sit there unused while traffic flows in the clear or fails silently. Verify the handshake path **end-to-end** (capture the actual ClientHello → Finished exchange), not just the presence of credentials.

12. **Dev-mode auth bypass shipped:** When no JWKS / signing keys are configured, a northbound API may **disable authentication entirely** and treat every request as a default/anonymous tenant. Convenient in development, **dangerous if shipped** — the absence of keys becomes an open door. Fail closed: no keys → reject, never default-allow.

13. **DPoP no-op validator:** A DPoP (RFC 9449) validator that parses the embedded JWK but then verifies the proof signature against an **empty key** will **always succeed**, silently disabling token-theft (proof-of-possession) protection. DPoP proofs **must** be verified against the JWK embedded in the proof header — confirm a tampered proof is actually rejected.

14. **Tenant isolation as a data-plane invariant:** Authenticating a tenant from a JWT claim or URI and then returning the **global device list** is a **P0 data leak**. Authentication identifies *who* is asking; it does not scope *what* they see. **Every device-scoped read must be filtered by tenant ownership** at the data layer. (See [northbound-nbi.md](northbound-nbi.md).)

15. **Entropy on constrained devices:** A DTLS RNG that emits predictable bytes when **no hardware entropy source is wired** must **fail closed**, not proceed with a weak key. A handshake that completes with low-entropy ephemeral keys is worse than a handshake that refuses to start.

---

## Northbound/API security

Northbound/API security — OAuth 2.1 + DPoP (RFC 9449), RFC 9457 problem-details error responses, and server-side RBAC as distinct from the device-side Access Control Object (/2) — is covered in [northbound-nbi.md](northbound-nbi.md). The device-plane controls in this document (DTLS/OSCORE, Security Object /0, Access Control /2) protect the southbound; they do **not** substitute for authenticating and tenant-scoping the management API.
