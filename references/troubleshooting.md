# LwM2M Troubleshooting Guide

## Table of Contents
1. [Systematic Diagnosis Approach](#systematic-diagnosis-approach)
2. [Bootstrap Failures](#bootstrap-failures)
3. [Registration Failures](#registration-failures)
4. [DTLS Handshake Failures](#dtls-handshake-failures)
5. [Observation & Notification Issues](#observation--notification-issues)
6. [FOTA Failures](#fota-failures)
7. [Queue Mode & Sleepy Device Issues](#queue-mode--sleepy-device-issues)
8. [Content-Format Negotiation Failures](#content-format-negotiation-failures)
9. [CID-Specific Issues](#cid-specific-issues)
10. [Wireshark Diagnosis Recipes](#wireshark-diagnosis-recipes)
11. [Quick-Reference Error Code Table](#quick-reference-error-code-table)

---

## Systematic Diagnosis Approach

For any LwM2M failure, work through these layers in order:

```
Layer 1: Network     → Can device reach the server IP/port? (ping, traceroute)
Layer 2: Transport   → Is UDP/TCP connectivity working? (packet capture)
Layer 3: Security    → Does the DTLS/TLS handshake complete? (Wireshark DTLS)
Layer 4: CoAP        → Is the CoAP request well-formed? (response code?)
Layer 5: LwM2M       → Is the LwM2M operation semantically correct? (payload, URI)
Layer 6: Application → Is the object/resource implementation correct? (callbacks)
```

**Key question at each layer:** "Does the expected response arrive, and what does it say?"

---

## Bootstrap Failures

### Symptom: Client-Initiated Bootstrap gets no response

| Check | Diagnosis | Fix |
|-------|-----------|-----|
| DNS resolution | Bootstrap-Server URI not resolving | Verify `coaps://bs.example.com:5684` resolves; use IP if DNS unavailable |
| Port reachability | UDP 5684 blocked by firewall/NAT | Check firewall rules; verify with `nc -u -z <host> 5684` |
| DTLS handshake | Fails before Bootstrap-Request | See [DTLS Handshake Failures](#dtls-handshake-failures) |
| Endpoint name | `ep` parameter rejected | Ensure Endpoint Client Name matches BS-Server provisioning database |
| Wrong URI path | POST to `/rd` instead of `/bs` | Bootstrap uses `/bs`, not `/rd` |

### Symptom: Bootstrap-Finish received but Registration fails

```
Diagnosis flow:
  Bootstrap-Finish received (2.04) ✓
  └── Client attempts DTLS to Server URI from /0/1/0
      ├── DTLS fails? → Credentials from bootstrap are wrong
      │     Check: /0/1/3 (PSK ID) and /0/1/5 (PSK key) match server config
      ├── Registration rejected (4.03)? → ACL issue or unknown endpoint
      └── No response? → Server URI/port wrong in /0/1/0
```

### Symptom: Bootstrap-Pack-Request (v1.2+) returns 4.00

- Client sent POST to `/bspack` but server doesn't support v1.2+
- Or the `ep` query parameter is missing
- Fallback: use standard client-initiated bootstrap (POST `/bs`)

---

## Registration Failures

### Symptom: POST /rd returns 4.00 Bad Request

| Cause | Diagnosis | Fix |
|-------|-----------|-----|
| Missing `ep` parameter | `ep` is mandatory in Uri-Query | Add `ep=<endpoint-name>` to registration query |
| Malformed link-format | Payload not valid CoRE Link Format | Validate `</0/0>,</1/0>,</3/0>` syntax; no trailing comma |
| Invalid `lwm2m` version | Server doesn't recognise version string | Use `lwm2m=1.2` not `lwm2m=v1.2` or `lwm2m=1.2.2` |
| Invalid binding | `b=X` where X is unsupported | Use standard values: `U`, `UQ`, `T`, `TQ`, `S`, `N`, `NQ`, `M`, `H` |

### Symptom: POST /rd returns 4.03 Forbidden

- Server does not recognise the endpoint or its credentials
- Client authenticated with bootstrap credentials instead of operational credentials
- Multi-tenant server: client assigned to wrong tenant/realm

### Symptom: Registration succeeds but Update fails (4.04)

```
Diagnosis flow:
  Registration: POST /rd → 2.01 Created, Location: /rd/abc123 ✓
  ... time passes ...
  Update: POST /rd/abc123 → 4.04 Not Found ✗
  │
  └── Cause: Registration expired (lifetime exceeded)
      ├── Client lifetime (lt) too short for sleep cycle
      ├── Server-side cleanup happened before update
      └── Fix: Increase lt, or wake earlier (80% of lt)
```

### Symptom: Registration Update returns 4.00

- Sending changed parameters that aren't allowed in Update
- Adding unknown query parameters
- Fix: Only `lt`, `b`, `sms` can change in Update; full re-registration for other changes

---

## DTLS Handshake Failures

### Common mbedTLS Error Codes

| Error Code | Meaning | Likely Cause |
|------------|---------|-------------|
| -0x6900 (-26880) | `MBEDTLS_ERR_SSL_CONN_EOF` | Server closed connection / unreachable |
| -0x6800 (-26624) | `MBEDTLS_ERR_SSL_PEER_CLOSE_NOTIFY` | Server rejected credentials |
| -0x7780 (-30592) | `MBEDTLS_ERR_SSL_FATAL_ALERT_MESSAGE` | Fatal TLS alert (bad cert, bad PSK) |
| -0x7200 (-29184) | `MBEDTLS_ERR_SSL_HANDSHAKE_FAILURE` | Cipher suite mismatch or PSK identity unknown |
| -0x2700 (-9984) | `MBEDTLS_ERR_X509_CERT_VERIFY_FAILED` | Certificate validation failed |

### PSK Handshake Failure Diagnosis

```
Step 1: Verify PSK identity matches server database
  Client sends: PSK Identity = "device001"
  Server looks up: "device001" → PSK key
  If identity unknown → ServerHello not sent → handshake times out

Step 2: Verify PSK key bytes match
  Common mistake: PSK stored as hex string vs raw bytes
  "AABBCCDD" (8 chars) ≠ 0xAA 0xBB 0xCC 0xDD (4 bytes)

Step 3: Verify cipher suite overlap
  Client offers: TLS_PSK_WITH_AES_128_CCM_8
  Server supports: TLS_PSK_WITH_AES_128_CBC_SHA256
  No overlap → handshake fails
```

### x509 Certificate Failure Diagnosis

```
Step 1: Certificate expired?
  openssl x509 -in client.crt -noout -dates

Step 2: CA trust chain valid?
  Server must trust the CA that signed the client certificate
  openssl verify -CAfile ca.crt client.crt

Step 3: SNI configured? (required v1.2+)
  Client must send SNI matching server's expected hostname
  Without SNI → server can't select correct certificate

Step 4: Key usage extensions correct?
  Client certificate needs: Digital Signature, Key Agreement
  Server certificate needs: Digital Signature, Key Encipherment
```

### CID Extension Mismatch (Type 53 vs 54)

See [CID-Specific Issues](#cid-specific-issues) below.

---

## Observation & Notification Issues

### Symptom: Observe setup succeeds but no notifications arrive

| Check | Diagnosis | Fix |
|-------|-----------|-----|
| pmin too high | pmin=3600 means minimum 1 hour between notifications | Lower pmin for more responsive notifications |
| Value not changing | st (step) threshold not exceeded | Reduce `st` value, or wait for larger change |
| CON vs NON | NON notifications may be dropped by lossy network | Set `con=true` (v1.2+) for reliable delivery |
| DTLS session lost | NAT rebinding broke the session | Enable CID (RFC 9146) |
| Token mismatch | Client restarted and lost observation state | Server should detect and re-establish observation |

### Symptom: Notification storm (too many notifications)

```
Diagnosis:
  pmin=0 + st=0 + rapidly changing sensor value
  = notification on every tiny value change
  
Fix:
  Set pmin ≥ 10 (minimum 10 seconds between notifications)
  Set st ≥ meaningful threshold (e.g., 0.5°C for temperature)
  Use Composite Observe (v1.2+) to batch multiple resources
```

### Symptom: Observation cancelled unexpectedly

- Server sent RST in response to a notification → observation cancelled
- Server de-registered the observation (Cancel Observation)
- Client restarted → all observations lost (server must re-establish)
- Registration expired → all observations invalidated

---

## FOTA Failures

### Object 5 Update Result Codes

| Result | Meaning | Common Cause |
|--------|---------|-------------|
| 0 | Initial value | Update not yet attempted |
| 1 | Success | Firmware updated successfully |
| 2 | Not enough flash | Firmware image too large for storage partition |
| 3 | Out of RAM | Insufficient RAM for decompression/validation |
| 4 | Connection lost during download | Network issue; implement Block2 resume logic |
| 5 | Integrity check failure | CRC/SHA mismatch — corrupted download or wrong image |
| 6 | Unsupported package type | Image format not recognised (wrong SoC variant?) |
| 7 | Invalid URI | Package URI (/5/0/1) malformed or unreachable |
| 8 | Firmware update failed | Apply/flash step failed — device rolled back (or bricked) |
| 9 | Unsupported protocol | URI scheme not supported (e.g., `https://` on CoAP-only client) |

### Download Interrupted — Resume Logic

```
Problem: Device enters PSM mid-download; Block2 transfer breaks

Resume strategy:
  1. Persist last successfully received block number to NVM
  2. On wake, resume GET with Block2 option starting at (last_block + 1)
  3. Server/CDN must support random-access Block2 (most do)
  4. If ETag changed → image updated on server → restart from block 0
  
  Example:
  Client → GET /firmware.bin, Block2: 42/0/1024  (resume at block 42)
  Server → 2.05 Content, Block2: 42/1/1024
  ... continues from block 42 ...
```

### Rollback After Failed Update

```
Strategy 1: Dual-bank (A/B) firmware slots
  ├── Bank A: current running firmware
  ├── Bank B: new firmware being downloaded/applied
  ├── On success: mark Bank B as active, reboot
  ├── On failure: hardware watchdog expires → boot from Bank A
  └── Safest approach, requires 2x flash for firmware

Strategy 2: Revert marker
  ├── Before applying: set NVM flag "update_in_progress"
  ├── After successful boot: clear flag, report Result=1
  ├── If flag still set at next boot → update failed → revert
  └── Less flash required, but more complex bootloader logic

Strategy 3: Delta/patch rollback (v2.0)
  ├── Store original firmware hash + delta patch
  ├── On failure: reverse-apply the delta patch
  └── Smallest flash footprint, but compute-intensive
```

### Multi-Component Update (v2.0)

```
Problem: Need to update modem FW, main FW, and app independently

Approach: Multiple /5 instances
  /5/0 → Main firmware (MCU image)
  /5/1 → Application layer (user code)
  /5/2 → Modem firmware (cellular stack)
  
  Each instance has its own:
  ├── Package URI (/5/x/1)
  ├── State machine (/5/x/3)
  ├── Update Result (/5/x/5)
  └── Can be updated independently (no full reflash)
```

---

## Queue Mode & Sleepy Device Issues

### Symptom: Device wakes but server doesn't send queued operations

| Check | Diagnosis | Fix |
|-------|-----------|-----|
| Registration expired | lt exceeded during sleep | Increase lt; wake before 80% of lt |
| DTLS session lost | NAT rebinding + no CID | Enable CID (RFC 9146); persist session to NVM |
| Server queue flushed | Server has max queue depth; oldest ops dropped | Increase server queue limit; prioritise critical ops |
| Awake window too short | Device sleeps before server finishes sending | Increase awake window timeout (30-60 seconds) |

### Symptom: Full DTLS re-handshake on every wake

```
Diagnosis flow:
  Device wakes → sends Registration Update
  Server responds with DTLS alert or no response
  Client falls back to full handshake (5+ RTT, ~2KB)
  
Causes:
  ├── CID not enabled → NAT rebinding breaks 5-tuple session match
  ├── CID enabled but extension type mismatch (53 vs 54)
  ├── Session material not persisted to NVM before sleep
  ├── Server evicted session from cache (too many concurrent sessions)
  └── DTLS epoch/sequence number rolled over
  
Fix priority:
  1. Enable CID with extension type 54 (RFC 9146 final)
  2. Persist DTLS session (CIDs, master secret, epoch, seq) to NVM
  3. Increase server session cache size
```

---

## Content-Format Negotiation Failures

### Symptom: 4.06 Not Acceptable

```
Server requested Accept: SenML-CBOR (112)
Client only supports TLV (11542) — it's a v1.0 client

Fix: Server should negotiate based on client's lwm2m version:
  lwm2m=1.0 → TLV (11542) or JSON (11543)
  lwm2m=1.1 → + SenML-JSON (110), SenML-CBOR (112)
  lwm2m=1.2 → + LwM2M CBOR (11543)
```

### Symptom: 4.15 Unsupported Content-Format

```
Client sent payload in a format the server doesn't understand

Common causes:
  ├── Client sent LwM2M CBOR to a v1.0/1.1 server
  ├── Wrong content-format number in the header
  ��── Composite operation payload not in SenML or LwM2M CBOR
  └── Single resource sent in TLV instead of plain text
```

---

## CID-Specific Issues

### CID Negotiation Silently Fails

```
Symptom: CID requested in ClientHello but not echoed in ServerHello
         Session works but falls back to 5-tuple binding
         After NAT rebinding → session breaks

Causes:
  1. Extension type mismatch:
     Client sends type 53 (obsolete draft) ← TinyDTLS, old mbedTLS
     Server expects type 54 (RFC 9146 final)
     Server ignores unknown extension → no CID
     
  2. Server doesn't support CID at all (e.g., OpenSSL 3.x)
  
  3. Zero-length CID requested:
     Client sends connection_id extension with length=0
     This means "I support CID but don't need one"
     Defeats NAT traversal purpose

Diagnosis:
  tshark -r capture.pcap -V -Y "dtls.handshake.type == 1" | grep -i "connection_id"
  Look for: Extension type (53 or 54) and CID value length
```

### CID Routing Failure in Cluster

```
Symptom: After NAT rebinding, CID-bearing packet reaches wrong server node
         Server node doesn't have the session → DTLS alert

Fix: Configure load balancer to inspect CID prefix in DTLS record header
     CID format: [2-byte node-ID prefix][N-byte session-ID]
     Route based on prefix → packet always reaches the correct node
```

---

## Wireshark Diagnosis Recipes

### Recipe 1: Capture Full Bootstrap + Registration Sequence

```bash
# Capture on CoAPs port (5684) for DTLS
tshark -i eth0 -f "udp port 5684" -w lwm2m_full.pcap

# Filter for bootstrap exchanges
tshark -r lwm2m_full.pcap -Y 'coap.opt.uri_path == "bs"' -T fields \
  -e frame.number -e ip.src -e coap.code -e coap.opt.uri_query

# Filter for registration exchanges
tshark -r lwm2m_full.pcap -Y 'coap.opt.uri_path == "rd"' -T fields \
  -e frame.number -e ip.src -e coap.code -e coap.opt.uri_query \
  -e coap.opt.uri_path
```

### Recipe 2: Diagnose DTLS Handshake

```bash
# Show all DTLS handshake messages with types
tshark -r capture.pcap -Y "dtls.handshake" -T fields \
  -e frame.number -e frame.time_relative -e ip.src -e ip.dst \
  -e dtls.handshake.type -e dtls.record.content_type \
  -E header=y -E separator='|'

# Handshake type values:
# 1=ClientHello, 2=ServerHello, 11=Certificate, 12=ServerKeyExchange
# 13=CertificateRequest, 14=ServerHelloDone, 15=CertificateVerify
# 16=ClientKeyExchange, 20=Finished
```

### Recipe 3: Track CID Usage Across NAT Rebinding

```bash
# Show CID values in handshake (negotiation)
tshark -r capture.pcap -V -Y "dtls.handshake.type == 1 || dtls.handshake.type == 2" \
  | grep -A2 "connection_id"

# Show CID-bearing records after handshake (ContentType 25)
tshark -r capture.pcap -Y "dtls.record.content_type == 25" -T fields \
  -e frame.number -e frame.time_relative -e ip.src -e udp.srcport \
  -E header=y -E separator='|'

# Compare source IP/port before and after sleep — CID should persist
```

### Recipe 4: Diagnose Notification Issues

```bash
# Show all Observe-related CoAP messages
tshark -r capture.pcap -Y "coap.opt.observe" -T fields \
  -e frame.number -e frame.time_relative -e ip.src \
  -e coap.code -e coap.opt.observe -e coap.token \
  -E header=y -E separator='|'

# observe=0 → Register observation
# observe=1 → Deregister observation
# observe=N (incrementing) → Notification sequence number
```

### Recipe 5: Diagnose FOTA Download

```bash
# Track Block2 transfer progress
tshark -r capture.pcap -Y "coap.opt.block2" -T fields \
  -e frame.number -e frame.time_relative -e ip.src \
  -e coap.code -e coap.opt.block2 \
  -E header=y -E separator='|'

# Block2 format: NUM/M/SZX
# NUM = block number, M = more blocks (1=yes, 0=last), SZX = size exponent
# Look for gaps in block numbers → lost/retransmitted blocks
```

---

## Quick-Reference Error Code Table

### CoAP Response Codes → LwM2M Meaning

| Code | Name | What It Means in LwM2M | Action |
|------|------|----------------------|--------|
| 2.01 | Created | Register/Create succeeded | OK |
| 2.02 | Deleted | De-register/Delete succeeded | OK |
| 2.04 | Changed | Update/Write/Execute succeeded | OK |
| 2.05 | Content | Read/Observe response | OK — check payload |
| 2.31 | Continue | Blockwise: more blocks to send | Send next block |
| 4.00 | Bad Request | Malformed request | Check URI, query params, payload |
| 4.01 | Unauthorized | DTLS authentication failed | Check credentials |
| 4.03 | Forbidden | No ACL permission | Check Access Control /2 |
| 4.04 | Not Found | Resource/Registration doesn't exist | Check path; registration may have expired |
| 4.05 | Method Not Allowed | Operation not permitted | E.g., Execute on read-only resource |
| 4.06 | Not Acceptable | Content-format not supported | Negotiate compatible format |
| 4.08 | Request Entity Incomplete | Missing blockwise block | Retransmit the missing block |
| 4.15 | Unsupported Content-Format | Payload format unknown | Use format compatible with client version |
| 5.00 | Internal Server Error | Server-side failure | Check server logs |
| 5.03 | Service Unavailable | Server overloaded | Retry with exponential backoff |
