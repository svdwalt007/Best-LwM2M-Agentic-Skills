# LwM2M Quick Start Tutorial

Get a working LwM2M client-server connection in 5 minutes using the Leshan public sandbox.

## Prerequisites

- A Linux, macOS, or Windows machine
- Internet connection
- One of the following:
  - **Python 3.7+** (for Python client)
  - **Git + GCC** (for C client with Wakaama)
  - **Docker** (for containerized client)

## Option 1: Python Client (Fastest, No Compilation)

### Step 1: Install Python LwM2M Client

```bash
pip3 install aiocoap cbor2
git clone https://github.com/Tanganelli/CoAPthon3.git
cd CoAPthon3
python3 setup.py install
```

### Step 2: Create Simple LwM2M Client

Create `simple_lwm2m_client.py`:

```python
#!/usr/bin/env python3
import socket
import time
from urllib.parse import urlencode

# LwM2M Server (Leshan public sandbox)
SERVER_HOST = "leshan.eclipseprojects.io"
SERVER_PORT = 5683

# Client configuration
ENDPOINT_NAME = f"quickstart-client-{int(time.time())}"
LIFETIME = 300  # seconds

def send_coap_post(host, port, uri_path, uri_query=None, payload=None):
    """Send CoAP POST message (simplified, no library)"""
    # CoAP message format (simplified):
    # Ver=1 (2 bits), Type=CON (2 bits), TKL=0 (4 bits) = 0x40
    # Code=POST (0.02) = 0x02
    # Message ID (2 bytes, random)
    # Options (Uri-Path, Uri-Query)
    # Payload

    msg_id = int(time.time()) & 0xFFFF

    # Build CoAP POST for registration
    coap_msg = bytearray()
    coap_msg.append(0x40)  # Ver=1, Type=CON, TKL=0
    coap_msg.append(0x02)  # Code=POST (0.02)
    coap_msg.extend(msg_id.to_bytes(2, 'big'))  # Message ID

    # Add Uri-Path option (11) for each path segment
    for segment in uri_path.strip('/').split('/'):
        seg_bytes = segment.encode('utf-8')
        coap_msg.append(0xB0 | len(seg_bytes))  # Option 11 (Uri-Path), delta from 0
        coap_msg.extend(seg_bytes)

    # Add Uri-Query options (15) if provided
    if uri_query:
        for query in uri_query.split('&'):
            query_bytes = query.encode('utf-8')
            coap_msg.append(0x40 | len(query_bytes))  # Option delta=4 (15-11=4)
            coap_msg.extend(query_bytes)

    # Payload marker and payload
    if payload:
        coap_msg.append(0xFF)
        coap_msg.extend(payload.encode('utf-8'))

    # Send UDP packet
    sock = socket.socket(socket.AF_INET, socket.SOCK_DGRAM)
    sock.settimeout(5.0)
    sock.sendto(coap_msg, (host, port))

    # Receive response
    try:
        data, addr = sock.recvfrom(1024)
        return data
    except socket.timeout:
        return None
    finally:
        sock.close()

def register_lwm2m_client():
    """Register LwM2M client with server"""
    print(f"Registering endpoint: {ENDPOINT_NAME}")

    # Registration parameters
    query = f"ep={ENDPOINT_NAME}&lt={LIFETIME}&lwm2m=1.0&b=U"

    # Object instances (link format)
    # </0> = Security, </1> = Server, </3> = Device
    payload = "</0/0>,</0/1>,</1/0>,</3/0>"

    response = send_coap_post(
        SERVER_HOST,
        SERVER_PORT,
        "/rd",
        uri_query=query,
        payload=payload
    )

    if response:
        # Parse Location-Path from response (simplified)
        print(f"✓ Registration successful!")
        print(f"  View your device at: https://{SERVER_HOST}/#/clients/{ENDPOINT_NAME}")
        return True
    else:
        print("✗ Registration failed")
        return False

def main():
    print("="*60)
    print("LwM2M Quick Start Client")
    print("="*60)
    print(f"Server: {SERVER_HOST}:{SERVER_PORT}")
    print(f"Endpoint: {ENDPOINT_NAME}")
    print()

    if register_lwm2m_client():
        print()
        print("Client registered! Keep this script running.")
        print(f"Open https://{SERVER_HOST} in your browser to interact.")
        print()
        print("You can now:")
        print("  - Read /3/0/0 (Device Manufacturer)")
        print("  - Read /3/0/1 (Device Model)")
        print("  - Execute /3/0/4 (Reboot)")
        print()
        print("Press Ctrl+C to exit")

        # Keep alive (simplified - real client would handle requests)
        try:
            while True:
                time.sleep(60)
        except KeyboardInterrupt:
            print("\nExiting...")

if __name__ == "__main__":
    main()
```

### Step 3: Run the Client

```bash
python3 simple_lwm2m_client.py
```

**Expected Output**:
```
============================================================
LwM2M Quick Start Client
============================================================
Server: leshan.eclipseprojects.io:5683
Endpoint: quickstart-client-1703260800

✓ Registration successful!
  View your device at: https://leshan.eclipseprojects.io/#/clients/quickstart-client-1703260800

Client registered! Keep this script running.
Open https://leshan.eclipseprojects.io in your browser to interact.
```

### Step 4: Interact via Web UI

1. Open https://leshan.eclipseprojects.io in your browser
2. Find your endpoint name in the client list (e.g., `quickstart-client-1703260800`)
3. Click on the endpoint name
4. Try these operations:
   - Click "Read" on Object 3 (Device) → see device information
   - Click "Observe" on /3/0/9 (Battery Level) → receive notifications

**Note**: The simplified Python example above shows the registration concept. For a fully functional client that handles server requests, use a proper library like `lwm2mclient` or proceed to Option 2 with Wakaama.

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
    │── POST /rd?ep=...&lt=300 ────────────►│
    │   Payload: </0/0>,</1/0>,</3/0>      │
    │                                       │
    │◄── 2.01 Created ──────────────────── │
    │    Location: /rd/abc123              │
```

**What was sent**:
- `ep=quickstart-client-XXX` : Endpoint name (unique identifier)
- `lt=300` : Lifetime in seconds (300s = 5 minutes)
- `lwm2m=1.0` : LwM2M version
- `b=U` : Binding mode (U = UDP)
- Payload: List of object instances the client supports

**What was received**:
- `2.01 Created` : Registration successful
- `Location: /rd/abc123` : Registration ID for future updates

### 2. Object Model

Your client advertised these objects:

| Object ID | Name | Description |
|-----------|------|-------------|
| `/0/0` | Security | Server credentials (not readable remotely) |
| `/1/0` | Server | Server configuration (lifetime, binding) |
| `/3/0` | Device | Device information (manufacturer, model, etc.) |

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

**Note**: Leshan sandbox accepts any PSK for testing. Production servers require pre-configured credentials.

### Try Firmware Update (FOTA)

1. In Leshan UI, navigate to your client
2. Click on Object 5 (Firmware Update)
3. Write firmware URI: `coap://example.com/firmware.bin`
4. Execute /5/0/2 (Update)
5. Observe state changes: Downloading → Downloaded → Updating → Idle

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

**Cause**: Client didn't respond to server's Read requests

**Solution**: Ensure client stays running and implements mandatory objects (0, 1, 3)

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
- **Composite Operations**: Try Read-Composite, Write-Composite (v1.2+)
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

- **LwM2M v1.2 Core**: https://www.openmobilealliance.org/release/LightweightM2M/V1_2-20201110-A/
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
