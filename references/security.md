# LwM2M Security — Comprehensive Reference

## Table of Contents
1. [Security Architecture Overview](#security-architecture-overview)
2. [Security Modes](#security-modes)
3. [DTLS 1.2 with Connection ID (RFC 9146)](#dtls-12-with-connection-id-rfc-9146)
4. [DTLS 1.3 (RFC 9147)](#dtls-13-rfc-9147)
5. [TLS for TCP Bindings](#tls-for-tcp-bindings)
6. [OSCORE (RFC 8613)](#oscore-rfc-8613)
7. [Credential Provisioning](#credential-provisioning)
8. [Access Control](#access-control)
9. [DTLS Library Comparison](#dtls-library-comparison)
10. [Session Persistence & NVM](#session-persistence--nvm)
11. [Common Security Pitfalls](#common-security-pitfalls)

---

## Security Architecture Overview

LwM2M security operates at three layers:

1. **Transport security:** DTLS (UDP), TLS (TCP), or OSCORE (any transport)
2. **Credential management:** Security Object (/0) stores per-server credentials
3. **Authorization:** Access Control Object (/2) enforces per-object per-server permissions

All LwM2M communication must use mutual authentication (per spec mandate). NoSec (Security Mode 3) is only for development/testing environments.

---

## Security Modes

### Mode 0: Pre-Shared Key (PSK)
- Simplest to deploy, lowest overhead
- Symmetric key shared between client and server
- Security Object /0 resources: Resource 3 = PSK Identity (String), Resource 5 = PSK Value (Opaque)
- DTLS cipher suites: TLS_PSK_WITH_AES_128_CCM_8 (mandatory), TLS_PSK_WITH_AES_128_CBC_SHA256
- Suitable for constrained devices with factory-provisioned keys
- Limitation: key rotation requires re-provisioning (bootstrap or out-of-band)

### Mode 1: Raw Public Key (RPK)
- Asymmetric keys without full certificate infrastructure
- Each side presents a raw public key (SubjectPublicKeyInfo structure)
- Security Object /0: Resource 3 = Client public key (RPK), Resource 5 = Client private key
- Lower overhead than x509 (no certificate chain validation)
- DTLS cipher suites: TLS_ECDHE_ECDSA_WITH_AES_128_CCM_8
- Trust established by pre-provisioning the expected public key

### Mode 2: x509 Certificate
- Full PKI with certificate chains
- Security Object /0: Resource 3 = Client certificate, Resource 4 = Server's CA cert or trust anchor, Resource 5 = Client private key
- Certificate validation: path validation, expiry, revocation
- SNI (RFC 6066) required from v1.2 for certificate-based security
- DTLS cipher suites: TLS_ECDHE_ECDSA_WITH_AES_128_CCM_8
- Highest overhead but best for large-scale managed deployments

### Mode 3: NoSec
- No transport security (plain CoAP)
- Development and testing only
- Not permitted for production deployments per spec

### Mode 4: EST over CoAP (v1.2+)
- Enrollment over Secure Transport using CoAP (not HTTP)
- Dynamic certificate provisioning without out-of-band key exchange
- Client uses initial credentials (PSK or manufacturer certificate) to authenticate to EST server
- EST server issues a new operational certificate
- Enables zero-touch provisioning with certificate rotation

---

## DTLS 1.2 with Connection ID (RFC 9146)

### Purpose
CID solves the NAT rebinding problem for sleepy devices. When a device enters PSM sleep and later wakes with a new source IP/port, the DTLS session would normally be lost (5-tuple mismatch). CID allows the session to survive address changes.

### Negotiation Flow
```
Client                                          Server
  │                                                │
  │── ClientHello ──────────────────────────────►  │
  │   + connection_id extension (CID_C, len>0)     │
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

### CID Rules
- Client MUST send `connection_id` extension with non-zero-length CID value
- Zero-length CID = "I support CID but don't need one from you" — defeats the purpose for sleepy devices
- Server MUST echo `connection_id` extension with its own CID in ServerHello
- If server omits the extension → CID not negotiated, 5-tuple binding only
- CID-bearing records use ContentType 25 (`tls12_cid`) instead of 23 (`application_data`)
- CID is placed in the record header, before the encrypted payload
- CID length is negotiated during handshake and fixed for the session

### TLS Extension Type Values: 53 vs 54

**This is a common source of confusion.** The IANA TLS ExtensionType registry has two entries related to CID:

- **Extension Type 53** — Was allocated via early allocation for an **earlier draft** version of the CID specification (draft-ietf-tls-dtls-connection-id). This draft used a different wire format that is **incompatible** with the final RFC 9146. Some older implementations (notably earlier TinyDTLS builds and some mbedTLS versions) may still use extension type 53.
- **Extension Type 54 (`connection_id`)** — The **final, standards-track allocation** per RFC 9146. This is the value that MUST be used for conformant implementations. The extension is marked DTLS-Only=Y and Recommended=N (because it targets specific use cases, not general TLS).

**Interop implications:** If a client sends extension type 53 and the server expects 54 (or vice versa), the CID negotiation silently fails — the server simply does not echo the extension in ServerHello, and the session falls back to 5-tuple binding. This is a known cause of CID failures in mixed-version deployments.

**ContentType 25 (`tls12_cid`):** Registered by RFC 9146 in the TLS ContentType registry. When CID is active, all encrypted records use ContentType 25 instead of 23 (`application_data`). The CID field immediately follows the record header, before the encrypted payload. The receiver uses the CID value to look up the security association, bypassing the 5-tuple entirely.

### Return Routability Check (RFC 9853)

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

### Key Differences from DTLS 1.2
- Reduced handshake round-trips (1-RTT, 0-RTT resumption)
- No ChangeCipherSpec messages
- Handshake messages encrypted earlier
- Built-in CID support (different from RFC 9146 approach)
- Improved key schedule using HKDF
- Forward secrecy mandatory (no static RSA key exchange)

### LwM2M Considerations
- DTLS 1.3 CID mechanism differs from DTLS 1.2 CID (RFC 9146)
- Server must support both DTLS 1.2 and 1.3 during migration
- Security Object /0 Resource 14 controls version constraints
- Not all DTLS libraries support 1.3 yet (see library comparison)

---

## TLS for TCP Bindings

For CoAP over TCP (RFC 8323), TLS is used instead of DTLS:

- TLS 1.2 (RFC 5246) or TLS 1.3 (RFC 8446)
- Same credential modes (PSK, RPK, x509, EST)
- SNI required for certificate mode (v1.2+)
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
Stored in Object 21 (OSCORE Object):

| Resource | Name | Purpose |
|----------|------|---------|
| 0 | OSCORE Master Secret | Shared secret for key derivation |
| 1 | Sender ID | Client's OSCORE identity |
| 2 | Recipient ID | Server's OSCORE identity |
| 3 | AEAD Algorithm | Encryption algorithm (AES-CCM-16-64-128 typical) |
| 4 | HMAC Algorithm | Key derivation HMAC |
| 5 | Master Salt | Optional salt for key derivation |

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

### EST over CoAP (Security Mode 4, v1.2+)
1. Client authenticates to EST server using initial credentials
2. EST server validates client identity
3. Client generates CSR (Certificate Signing Request)
4. EST server signs and returns operational certificate
5. Client stores certificate in Security Object /0
6. Client uses new certificate for server authentication

---

## Access Control

Object 2 (Access Control) provides per-Object-Instance authorization:

- Each Access Control Object Instance specifies permissions for one Object Instance
- Resource 2 (ACL) is multi-instance: one Resource Instance per server (keyed by Short Server ID)
- Permission bits: Read=1, Write=2, Execute=4, Create=8, Delete=16
- The Access Control Owner (Resource 3) has full permissions
- A server can only access object instances where it has the required permission bits

### Multi-Server Scenarios
When a client connects to multiple servers, Access Control is critical:
- Bootstrap-Server provisions Access Control instances
- Each server gets specific permissions per object instance
- A Read-only monitoring server might get only Read (1) permission
- A management server might get Read+Write+Execute (7)

---

## DTLS Library Comparison

| Feature | mbedTLS 3.x | TinyDTLS | wolfSSL 5.x | OpenSSL 3.x |
|---------|-------------|----------|-------------|-------------|
| DTLS 1.2 CID (RFC 9146) | ✅ (3.2+) | ⚠ (draft impl) | ✅ (5.4+) | ❌ |
| DTLS 1.3 | ❌ | ❌ | ⚠ (partial) | ✅ (3.2+) |
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

**OpenSSL 3.x:** No CID at all — unsuitable for sleepy devices. Too large for MCU targets. DTLS 1.3 experimental in 3.2+.

---

## Session Persistence & NVM

For sleepy devices using Queue Mode + CID, DTLS session material must survive sleep cycles:

### What to Persist (NVM)
- **Permanent:** PSK identity/key, certificates, private keys, Endpoint Client Name, SSID, Security Object Resources 0–18
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

### Pre-Deployment Security Checklist

Use this checklist before deploying LwM2M devices to production:

**Credential Management:**
- [ ] Per-device unique PSK keys (never reuse across devices)
- [ ] Bootstrap credentials separate from operational credentials
- [ ] PSK keys stored as raw bytes, not hex-encoded strings
- [ ] Certificate expiry dates monitored (x509 mode)
- [ ] CA trust chain validated end-to-end (x509 mode)
- [ ] EST over CoAP (Mode 4) evaluated for zero-touch provisioning
- [ ] Key rotation plan documented (how to re-provision credentials)

**DTLS Configuration:**
- [ ] CID enabled with non-zero-length value (RFC 9146)
- [ ] CID extension type 54 (not obsolete type 53)
- [ ] Extended Master Secret enabled (RFC 7627)
- [ ] SNI configured for certificate mode (required v1.2+)
- [ ] DTLS session material persisted to NVM before sleep
- [ ] Return Routability Check (RFC 9853) or CoAP Echo (RFC 9175) enabled
- [ ] Cipher suite restricted to minimum necessary (no weak ciphers)

**Server-Side:**
- [ ] CID-aware load balancer for clustered deployments
- [ ] Session cache sized for expected concurrent devices
- [ ] NoSec (Mode 3) disabled or restricted to test environments
- [ ] Certificate chain validation not skipped (even on constrained paths)
- [ ] Multi-tenant credential isolation enforced
- [ ] Rate limiting on DTLS handshakes (DoS protection)

**OSCORE (if used):**
- [ ] Sequence number persistence across reboots
- [ ] Sequence number recovery mechanism (RFC 8613 Appendix B.2)
- [ ] OSCORE context synchronisation tested for crash recovery

### Detailed Pitfalls

1. **Zero-length CID request:** Client sends CID extension with zero length. This signals CID support but asks the server not to use one — defeats the NAT traversal purpose. Always request a non-zero-length CID.

2. **NoSec in production:** Security Mode 3 must never be used in production. It provides no authentication or encryption.

3. **PSK key reuse:** Using the same PSK across multiple devices. If one device is compromised, all devices are compromised. Use per-device unique keys.

4. **No session persistence:** Not persisting DTLS session to NVM before sleep. Every wake cycle requires a full handshake (5+ round-trips, several KB of data).

5. **Missing SNI:** For x509 mode in v1.2+, SNI is required. Without it, certificate selection on multi-tenant servers fails.

6. **Missing Extended Master Secret:** RFC 7627 is recommended for DTLS 1.2. Without it, certain triple-handshake attacks are possible.

7. **Ignoring CID in cluster routing:** In multi-node server deployments, CID-bearing packets must be routed to the node holding the session. Without CID-aware routing, sessions break after NAT rebinding.

8. **Certificate chain validation skipped:** Some constrained implementations skip certificate path validation for performance. This defeats the purpose of x509 security.

9. **OSCORE context not synchronised:** If client and server OSCORE sequence numbers diverge (e.g., after a crash), replay protection blocks legitimate messages. Implement sequence number recovery per RFC 8613 Appendix B.2.

10. **Bootstrap credentials as operational credentials:** The bootstrap PSK/cert should be different from operational credentials. Using the same credentials for both roles weakens the security boundary.

11. **PSK stored as hex string instead of raw bytes:** A PSK of `"AABBCCDD"` (8 ASCII characters) is not the same as `0xAA 0xBB 0xCC 0xDD` (4 raw bytes). Ensure client and server agree on encoding. This is one of the most common deployment issues.

12. **No DTLS handshake rate limiting:** Without rate limiting, an attacker can exhaust server CPU with spoofed ClientHello messages. Implement per-IP handshake rate limits and use DTLS HelloVerifyRequest (cookie) to validate source addresses.

13. **Firmware images not signed:** Object 5 firmware packages should be cryptographically signed (e.g., ECDSA over SHA-256). Without signing, a compromised download server can push malicious firmware. The client must verify the signature before applying.

14. **ACL not provisioned for multi-server:** In multi-server deployments, if Access Control Object (/2) is not provisioned during bootstrap, all servers may get full access to all objects. Always provision explicit ACLs.
