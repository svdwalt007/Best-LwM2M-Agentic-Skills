# LwM2M Quick Start Tutorial

Get a working LwM2M client-server connection in 5 minutes using the Leshan public sandbox.

## Prerequisites

- A Linux, macOS, or Windows machine
- Internet connection
- One of the following:
  - **Python 3.10+** (for the Python client; aiocoap 0.4.17 requires >= 3.10)
  - **Git + GCC** (for C client with Wakaama)
  - **Docker** (for containerized client)

## Option 1: Python Client (Fastest, No Compilation)

### Step 1: Install aiocoap

```bash
python3 -m venv lwm2m-venv && . lwm2m-venv/bin/activate
pip install aiocoap   # tested with aiocoap 0.4.17 (requires Python >= 3.10)
```

aiocoap builds the CoAP header and options (delta encoding, extended option lengths, Content-Format) per RFC 7252 §3.1, so no CoAP bytes are hand-encoded.

### Step 2: Create Simple LwM2M Client

Create `simple_lwm2m_client.py`. It registers, sends a Registration Update every `lifetime/2` seconds, and De-registers on Ctrl+C:

```python
#!/usr/bin/env python3
"""Minimal LwM2M 1.2 Register / Update / De-register client (NoSec, UDP) using aiocoap."""
import asyncio, os, signal, sys, time
from datetime import datetime, timezone
from aiocoap import Context, Message, POST, DELETE

APP_VERSION = "1.0.0"
SERVER = "coap://leshan.eclipseprojects.io:5683"
ENDPOINT_NAME = f"quickstart-client-{int(time.time())}"
LIFETIME = 300  # seconds; must equal /1/0/1
LWM2M_VERSION = "1.1"  # Leshan sandbox answered 4.12 to lwm2m=1.2 (tested 2026-10-10); use "1.2" against a 1.2 server
# Objects 0 (Security), 21 (OSCORE) and 23 (COSE) are never advertised (Core TS 1.2.2 §6.2.1).
OBJECT_LINKS = b"</1/0>,</3/0>"
LINK_FORMAT = 40  # application/link-format

async def request(ctx, code, uri, payload=b"", content_format=None):
    msg = Message(code=code, uri=uri, payload=payload)
    if content_format is not None:
        msg.opt.content_format = content_format
    return await ctx.request(msg).response

async def main():
    print(f"LwM2M Quick Start Client v{APP_VERSION} | LwM2M {LWM2M_VERSION} over CoAP/UDP (RFC 7252)")
    print(f"Started {datetime.now(timezone.utc):%Y-%m-%d %H:%M:%S} UTC | server {SERVER} | ep {ENDPOINT_NAME}")
    print("Ctrl+C = graceful exit (De-register) | Ctrl+X = abrupt exit (no De-register)", flush=True)

    loop = asyncio.get_running_loop()
    stop = asyncio.Event()
    mode = {"deregister": True}
    for sig in (signal.SIGINT, signal.SIGTERM):
        loop.add_signal_handler(sig, stop.set)

    tty_saved = None
    if sys.stdin.isatty():  # Ctrl+X (0x18) detection; ISIG kept so Ctrl+C still raises SIGINT
        import termios
        fd = sys.stdin.fileno()
        tty_saved = termios.tcgetattr(fd)
        raw = termios.tcgetattr(fd)
        raw[3] &= ~(termios.ICANON | termios.ECHO)
        raw[6][termios.VMIN], raw[6][termios.VTIME] = 0, 0
        termios.tcsetattr(fd, termios.TCSANOW, raw)
        def on_key():
            if b"\x18" in os.read(fd, 64):
                mode["deregister"] = False
                stop.set()
        loop.add_reader(fd, on_key)

    ctx = await Context.create_client_context()
    try:
        # Register: POST /rd?ep=..&lt=..&lwm2m=1.2&b=U  (Transport TS 1.2.2 §6.4.3)
        query = f"ep={ENDPOINT_NAME}&lt={LIFETIME}&lwm2m={LWM2M_VERSION}&b=U"
        resp = await request(ctx, POST, f"{SERVER}/rd?{query}", OBJECT_LINKS, LINK_FORMAT)
        if not resp.code.is_successful():
            print(f"Registration failed: {resp.code} {resp.payload.decode(errors='replace')}")
            return
        location = "/" + "/".join(resp.opt.location_path)  # Location-Path options, e.g. rd, IETgKZzFOx
        print(f"Registered: {resp.code}, location {location}", flush=True)

        while not stop.is_set():
            try:  # Registration Update before lifetime expiry
                await asyncio.wait_for(stop.wait(), timeout=LIFETIME / 2)
            except asyncio.TimeoutError:
                resp = await request(ctx, POST, f"{SERVER}{location}")
                print(f"Update: {resp.code}", flush=True)

        if mode["deregister"]:
            resp = await request(ctx, DELETE, f"{SERVER}{location}")
            print(f"De-registered: {resp.code}")
        else:
            print("Abrupt exit: De-register skipped")
    finally:
        if tty_saved is not None:
            import termios
            loop.remove_reader(sys.stdin.fileno())
            termios.tcsetattr(sys.stdin.fileno(), termios.TCSANOW, tty_saved)
        await ctx.shutdown()

if __name__ == "__main__":
    asyncio.run(main())
```

The Register request this produces (Transport TS 1.2.2 §6.4.3, Table 6.4.3-2):

```
CON POST  Uri-Host: leshan.eclipseprojects.io
          Uri-Path: rd
          Uri-Query: ep=quickstart-client-1791613491
          Uri-Query: lt=300
          Uri-Query: lwm2m=1.1
          Uri-Query: b=U
          Content-Format: 40 (application/link-format)
          Payload: </1/0>,</3/0>
```

Equivalent one-shot construction: `Message(code=POST, uri='coap://leshan.eclipseprojects.io:5683/rd?ep=...&lt=300&lwm2m=1.2&b=U', payload=b'</1/0>,</3/0>', content_format=40)`. aiocoap splits the URI into one Uri-Path option (`rd`) and one Uri-Query option per `&`-separated parameter.

### Step 3: Run the Client

```bash
python3 simple_lwm2m_client.py
```

The signal and `termios` handling is POSIX-only: run it on Linux, macOS or WSL.

**Expected Output** (captured against the sandbox on 2026-10-10, with a shortened lifetime so the Updates appear quickly):
```
LwM2M Quick Start Client v1.0.0 | LwM2M 1.1 over CoAP/UDP (RFC 7252)
Started 2026-10-10 06:24:51 UTC | server coap://leshan.eclipseprojects.io:5683 | ep quickstart-client-1791613491
Ctrl+C = graceful exit (De-register) | Ctrl+X = abrupt exit (no De-register)
Registered: 2.01 Created, location /rd/IETgKZzFOx
Update: 2.04 Changed
Update: 2.04 Changed
De-registered: 2.02 Deleted
```

With `LWM2M_VERSION = "1.2"` the sandbox replied `4.12 Precondition Failed unsupported lwm2m version`, which is the response a server gives when it does not support the enabler version the client indicates (Core TS 1.2.2 §6.2.1: the server MUST refuse such a Registration). Use `lwm2m=1.2` against a server that implements LwM2M 1.2.

### Step 4: Interact via Web UI

1. Open https://leshan.eclipseprojects.io in your browser
2. Find your endpoint name in the client list (e.g., `quickstart-client-1791613491`)
3. Click on the endpoint name to see the registration (endpoint, lifetime, binding, advertised objects)
4. Press Ctrl+C in the client terminal and watch the entry disappear (De-register)

**Note**: This minimal client implements only the Registration interface (Register, Update, De-register). It does not serve Objects, so Read/Write/Execute/Observe from the web UI will time out. For a client that answers server requests, use Option 2 (Wakaama) or Option 3 (Leshan client).

---

## Option 2: C Client with Wakaama (Production-Ready)

### Step 1: Clone Wakaama

```bash
git clone https://github.com/eclipse/wakaama.git
cd wakaama
```

### Step 2: Build the Example Client

```bash
mkdir build && cd build
cmake ..
make lwm2mclient
```

### Step 3: Run Against Leshan Sandbox

```bash
./examples/client/lwm2mclient \
  -n quickstart-wakaama-client \
  -h leshan.eclipseprojects.io \
  -p 5683 \
  -4
```

**Command Options**:
- `-n` : Endpoint name
- `-h` : Server hostname
- `-p` : Server port (5683 for CoAP, 5684 for CoAPS)
- `-4` : Use IPv4

**Expected Output**:
```
New client #0 registered.
Device Object:
  /3/0/0: Manufacturer = "Open Mobile Alliance"
  /3/0/1: Model Number = "Lightweight M2M Client"
  /3/0/2: Serial Number = "345000123"
  /3/0/3: Firmware Version = "1.0"
Client registered successfully!
```

### Step 4: Interact via Web UI

1. Open https://leshan.eclipseprojects.io
2. Find `quickstart-wakaama-client` in the client list
3. Try operations:
   - **Read**: `/3/0/0` (Manufacturer) → "Open Mobile Alliance"
   - **Read**: `/3/0/9` (Battery Level) → 100
   - **Write**: `/3/0/13` (Current Time) → Unix timestamp
   - **Execute**: `/3/0/4` (Reboot) → client will restart
   - **Observe**: `/3/0/9` → receive periodic notifications

### Step 5: Explore Object Instances

In the Wakaama client terminal, you can:

```
# List available commands
help

# List all objects
list

# Change battery level (triggers observation notification)
change /3/0/9 85

# Add a new object instance (e.g., Temperature Sensor)
create /3303/0
```

---

## Option 3: Docker Client (No Local Setup)

### Step 1: Run Leshan Client Container

```bash
docker run -it --rm \
  --name lwm2m-quickstart \
  eclipse/leshan-client:latest \
  -n quickstart-docker-client \
  -u coap://leshan.eclipseprojects.io:5683
```

### Step 2: View in Browser

Open https://leshan.eclipseprojects.io and find `quickstart-docker-client`.

---

## Understanding What Just Happened

### 1. Registration Flow

```
Your Client                           Leshan Server
    │                                       │
    │── POST /rd?ep=...&lt=300&lwm2m=1.1&b=U ►│
    │   Payload: </1/0>,</3/0>             │
    │                                       │
    │◄── 2.01 Created ──────────────────── │
    │    Location-Path: rd                  │
    │    Location-Path: IETgKZzFOx          │
```

**What was sent** (Core TS 1.2.2 Table 6.2.1-1; query names per Transport TS 1.2.2 Table 6.4.3-1):
- `ep=quickstart-client-XXX` : Endpoint Client Name, unique on that server. Optional from LwM2M 1.1 when it equals the security-protocol identifier (e.g. PSK identity) and no proxy sits in between; the Register SHOULD still include it (Core TS 1.2.2 §6.2.1), and with NoSec there is no security identifier to fall back on, so keep it for the sandbox
- `lt=300` : Lifetime in seconds (300 s = 5 minutes); must equal `/1/x/1`
- `lwm2m=1.1` : LwM2M enabler version (`lwm2m=1.2` for a 1.2 client against a 1.2 server; an unsupported version is refused, the sandbox answers 4.12)
- `b=U` : Binding Mode (U = UDP). From LwM2M 1.1 Queue Mode is a separate valueless `Q` query parameter (e.g. `&b=U&Q`), not part of the binding string (the 1.0 `b=UQ` form is gone)
- Payload: CoRE Link Format list of supported Objects and Object Instances. Security (0), OSCORE (21) and COSE (23) Object Instances MUST NOT be listed (Core TS 1.2.2 §6.2.1); they are only visible to the Bootstrap-Server via Bootstrap-Discover

**What was received**:
- `2.01 Created` : Registration successful
- Location-Path options `rd`, `IETgKZzFOx` : the registration location `/rd/IETgKZzFOx`. The server MUST return a location under `/rd` (Transport TS 1.2.2 §6.4.3). Updates are `POST /rd/IETgKZzFOx`; De-register is `DELETE /rd/IETgKZzFOx`. In aiocoap: `resp.opt.location_path`

### 2. Object Model

Your client advertised these objects:

| Object ID | Name | Description |
|-----------|------|-------------|
| `/1/0` | Server | Server configuration (lifetime, binding) |
| `/3/0` | Device | Device information (manufacturer, model, etc.) |

Object 0 (Security) is present on every client but is never advertised in Register or Update, nor are Objects 21 (OSCORE) and 23 (COSE). Objects 0 and 21 are reported only to the Bootstrap-Server (Bootstrap-Discover) and are not accessible through the Device Management and Service Enablement interface (Core TS 1.2.2 §6.1.7.3).

### 3. Read Operation

When you clicked "Read" on `/3/0/0` in the web UI:

```
Leshan Server                         Your Client
    │                                       │
    │── GET /3/0/0 ───────────────────────►│
    │   (Read Manufacturer resource)         │
    │                                       │
    │◄── 2.05 Content ──────────────────── │
    │    Payload: "Open Mobile Alliance"    │
```

### 4. Observe Operation

When you clicked "Observe" on `/3/0/9` (Battery Level):

```
Leshan Server                         Your Client
    │                                       │
    │── GET /3/0/9 + Observe:0 ───────────►│
    │   (Start observing battery level)     │
    │                                       │
    │◄── 2.05 Content ──────────────────── │
    │    Observe:12345, Payload: 100        │
    │                                       │
    │   ... time passes, battery changes ... │
    │                                       │
    │◄── 2.05 Content (Notify) ─────────── │
    │    Observe:12346, Payload: 95         │
    │                                       │
    │◄── 2.05 Content (Notify) ─────────── │
    │    Observe:12347, Payload: 90         │
```

---

## Next Steps

### Add More Objects

Edit Wakaama client to add custom objects:

```bash
cd wakaama/examples/client
# Edit object_temperature.c (if exists) or create custom object
```

Example custom temperature sensor object:

```c
// Object 3303: Temperature Sensor
#define TEMPERATURE_OBJECT_ID   3303
#define RES_SENSOR_VALUE        5700
#define RES_SENSOR_UNITS        5701

static uint8_t prv_temperature_read(lwm2m_context_t *contextP,
                                     uint16_t instanceId,
                                     int *numDataP,
                                     lwm2m_data_t **dataArrayP,
                                     lwm2m_object_t *objectP)
{
    // Simulate temperature reading
    float temperature = 23.5;

    *numDataP = 1;
    *dataArrayP = lwm2m_data_new(1);
    (*dataArrayP)[0].id = RES_SENSOR_VALUE;
    lwm2m_data_encode_float(temperature, *dataArrayP);

    return COAP_205_CONTENT;
}
```

### Enable DTLS Security

Run Wakaama client with PSK:

```bash
./examples/client/lwm2mclient \
  -n secure-client \
  -h leshan.eclipseprojects.io \
  -p 5684 \
  -i my-psk-identity \
  -s 0123456789ABCDEF0123456789ABCDEF
```

**Note**: Before connecting, register the PSK identity and key for your endpoint on the sandbox Security page (vendor behaviour, verify against the current Leshan demo). Production servers require pre-provisioned credentials, normally delivered by bootstrap.

### Try Firmware Update (FOTA)

Use a client that implements Object 5 (e.g. the Wakaama `lwm2mclient`; the minimal Python client does not). Flow per Core TS 1.2.2 Appendix E.6 / E.6.1:

1. In Leshan UI, navigate to your client and open Object 5 (Firmware Update)
2. Observe `/5/0/3` (State) and `/5/0/5` (Update Result)
3. Write the Package URI `/5/0/1`: `coap://example.com/firmware.bin`. State goes Idle (0) → Downloading (1) → Downloaded (2)
4. Wait until `/5/0/3` = 2 (Downloaded). Update `/5/0/2` is only executable in Downloaded state
5. Execute `/5/0/2` (Update). State goes Downloaded (2) → Updating (3) → Idle (0) on success
6. Read `/5/0/5` (Update Result): 1 = firmware updated successfully. On failure the state returns to Idle or Downloaded depending on the cause (e.g. integrity check failure → Idle) and Update Result carries the reason (2–9); 10 = cancelled (Cancel executed), 11 = update deferred by the user, with State staying Downloaded

Writing an empty string to `/5/0/1` resets the state machine to Idle with Update Result 0.

### Run Your Own Server

Instead of the public sandbox, run Leshan server locally:

```bash
# Download Leshan server
wget https://ci.eclipse.org/leshan/job/leshan/lastSuccessfulBuild/artifact/leshan-server-demo.jar

# Run server
java -jar leshan-server-demo.jar

# Server runs at http://localhost:8080 (web UI) and coap://localhost:5683 (CoAP)
```

Point your client to localhost:

```bash
./lwm2mclient -n local-client -h localhost -p 5683 -4
```

---

## Common Issues

### Registration Fails

**Symptom**: `Connection refused` or timeout

**Solutions**:
- Check firewall: allow outbound UDP port 5683
- Verify server is reachable: `ping leshan.eclipseprojects.io`
- Try explicit IPv4: add `-4` flag to client command
- Check endpoint name is unique (use timestamp suffix)

### Client Disappears from UI

**Symptom**: Client shows up briefly then disappears

**Cause**: The registration lifetime (`lt`) expired without a Registration Update, so the server removed it

**Solution**: Ensure the client stays running and sends a Registration Update (`POST /rd/<location>`) before `lt` expires; implement the mandatory objects (0, 1, 3) so server requests are answered

### DTLS Handshake Fails

**Symptom**: `DTLS handshake failed` error

**Solutions**:
- Verify PSK identity and key match server configuration
- Check DTLS library is compiled in: `ldd lwm2mclient | grep dtls`
- Try NoSec mode first (port 5683) to verify CoAP works

### "Object Not Found" Error

**Symptom**: Server returns `4.04 Not Found`

**Cause**: Client didn't register the requested object

**Solution**: Check client registration payload includes the object ID

---

## Learning Resources

### Explore the Leshan Server

- **Client List**: See all connected clients
- **Object Browser**: Explore object model hierarchy
- **Request Viewer**: See CoAP messages in real-time
- **Composite Operations**: Try Read-Composite, Write-Composite (LwM2M 1.1+, together with Observe-Composite and Send)
- **Security**: Configure PSK, RPK, or x.509 credentials

### Wireshark Capture

Capture LwM2M traffic to understand the protocol:

```bash
sudo tcpdump -i any -w lwm2m.pcap port 5683
```

Open in Wireshark with CoAP dissector enabled.

### Read the Code

Explore Wakaama source code:

- `examples/client/lwm2mclient.c` : Main client loop
- `core/liblwm2m.c` : Core LwM2M logic
- `examples/client/object_device.c` : Device Object implementation
- `examples/client/object_firmware.c` : FOTA implementation

### Official Specifications

- **LwM2M v1.2.2 Core** (OMA-TS-LightweightM2M_Core-V1_2_2-20240613-A, Approved 2024-06-13) and Transport (OMA-TS-LightweightM2M_Transport-V1_2_2-20240613-A): https://www.openmobilealliance.org/release/LightweightM2M/V1_2_2-20240613-A/
- **CoAP RFC 7252**: https://www.rfc-editor.org/rfc/rfc7252.html
- **DTLS RFC 6347**: https://www.rfc-editor.org/rfc/rfc6347.html

---

## Congratulations!

You now have a working LwM2M client-server connection. You've learned:

- ✓ How LwM2M registration works
- ✓ The object model structure (/ObjectID/InstanceID/ResourceID)
- ✓ Basic operations (Read, Write, Execute, Observe)
- ✓ How to use the Leshan sandbox for testing

**Next**: Explore the comprehensive reference documentation in the `references/` directory to dive deeper into LwM2M protocol details, security modes, transport bindings, and advanced features.
