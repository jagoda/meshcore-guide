# Build Your Own Client

This page is a language-agnostic checklist for implementing a Companion API client from scratch, along with guidance on when to use an existing library instead.

## Start with a library if you can

Before implementing raw framing, check whether an official SDK covers your language:

| Language | Library | Notes |
|---|---|---|
| Python | [`meshcore_py`](https://github.com/meshcore-dev/meshcore_py) | Async, event-driven. Supports USB serial, BLE, TCP. |
| JavaScript / TypeScript | [`meshcore.js`](https://github.com/meshcore-dev/meshcore.js) | Browser and Node.js. |

If your target language has neither, or if you need a minimal embedded client without a full SDK dependency, implement the protocol directly using the steps below.

## Step 1: Choose your transport

| Transport | Best for | Key constraint |
|---|---|---|
| BLE | Mobile apps, single-board computers | Requires BLE stack; MTU negotiation needed for large frames |
| USB serial | Desktop tools, embedded Linux, always-on servers | Most reliable; add length-prefix framing (see below) |
| TCP / Wi-Fi | Remote home-automation, NAS, rack server | Same framing as serial; needs reachable IP on local network |

For server-side clients (home automation, logging, dashboards), **USB serial or TCP** is almost always the better choice — BLE adds reconnect complexity and power requirements.

## Step 2: Implement transport framing

### BLE

1. Scan for the NUS service UUID `6E400001-B5A3-F393-E0A9-E50E24DCCA9E`.
2. Discover and cache the RX (`6E400002…`) and TX (`6E400003…`) characteristics.
3. Subscribe to TX notifications before sending any commands.
4. **Request MTU 512** if your platform supports it. The default 20-byte BLE payload is too small for `CMD_SET_CHANNEL` (50-byte payload) and potentially for other commands.
5. Write companion protocol frames directly to the RX characteristic. One write = one frame.
6. Each TX notification delivers one complete companion protocol frame.

### Serial / TCP

Implement the length-prefix envelope described in [Frame Model — Serial transport framing](frame-model.md#serial-transport-framing):

```python
import struct

def encode_frame(payload: bytes) -> bytes:
    """Wrap a companion protocol frame for serial/TCP transmission."""
    return b'>' + struct.pack('<H', len(payload)) + payload

def decode_frames(stream_reader):
    """Yield companion protocol frames from a byte stream."""
    while True:
        header = stream_reader.read_exactly(3)
        if header[0] != ord('<'):
            # Out of sync — scan for next '<'
            continue
        length = struct.unpack('<H', header[1:3])[0]
        yield stream_reader.read_exactly(length)
```

The header byte `<` (0x3C) is the sync marker. If your read loop sees unexpected bytes before `<`, discard them — the protocol is self-resynchronising. After a complete frame is received, the parser returns to idle and looks for the next `<`.

## Step 3: Implement the command queue

The protocol is one-command-one-response. Never send a second command before the response to the first has arrived. See [Command Lifecycle — Command queue in practice](command-lifecycle.md#command-queue-in-practice) for a reference implementation.

Your queue must also handle **push codes** (unsolicited frames from the node) without stalling the command queue. The receive loop should:

```
frame[0] == 0x83 (MSG_WAITING) → trigger async message drain
frame[0] == 0x80 (ADVERT)      → update contact state
frame[0] == (expected response) → resolve pending command
otherwise                       → log and discard
```

## Step 4: Startup handshake

Your client is not usable until the startup sequence completes. Run these in order:

```python
# 1. Send CMD_APP_START (0x01), wait for PACKET_SELF_INFO (0x05)
self_info = await queue.send(b'\x01' + b'\x00'*7 + b'my-app', expect=0x05)

# 2. Send CMD_DEVICE_QUERY (0x16 0x03), wait for PACKET_DEVICE_INFO (0x0D)
device_info = await queue.send(b'\x16\x03', expect=0x0D)
max_channels = parse_max_channels(device_info)

# 3. Set device time
ts = struct.pack('<I', int(time.time()))
await queue.send(b'\x13' + ts, expect=0x00)  # 0x00 = PACKET_OK

# 4. Get contacts (multi-frame response)
contacts = await get_contacts(queue)

# 5. Get all channel slots
for idx in range(max_channels):
    channel_info = await queue.send(bytes([0x1F, idx]), expect=0x12)

# 6. Drain queued messages
while True:
    msg = await queue.send(b'\x0A', expect_any=True)
    if msg[0] == 0x0A:  # PACKET_NO_MORE_MSGS
        break
```

## Step 5: Handle the contact list protocol

`CMD_GET_CONTACTS` uses a start/items/end pattern rather than a single response. Your implementation needs to accumulate frames until it sees `PACKET_CONTACT_END` (`0x04`):

```python
async def get_contacts(queue):
    await queue.send_raw(b'\x05')  # CMD_GET_CONTACTS = 0x05
    contacts = []
    async for frame in queue.receive_until(0x04):  # stop at CONTACT_END
        if frame[0] == 0x02:  # PACKET_CONTACT_START
            pass
        elif frame[0] == 0x03:  # PACKET_CONTACT
            contacts.append(parse_contact(frame))
    return contacts
```

This multi-frame pattern is specific to contact enumeration. All other commands return exactly one response frame.

## Step 6: Auto-reconnect

Nodes disconnect after inactivity on BLE and may restart unexpectedly on all transports. Implement reconnect with **exponential backoff**:

```python
async def with_reconnect(connect_fn):
    delay = 1.0
    while True:
        try:
            await connect_fn()
            delay = 1.0   # reset on success
            return
        except ConnectionError:
            await asyncio.sleep(delay)
            delay = min(delay * 2, 60.0)
```

After reconnecting, re-run the full startup sequence — the firmware has no memory of your previous session.

## Gotchas checklist

- **SNR encoding.** SNR values (`last_snr` in stats, SNR byte in V3 message frames) are signed int8 multiplied by 4. Divide by `4.0` to recover dB. A raw byte value of `0xEC` (signed −20) means −5.0 dB, not −20 dB.

- **Endianness.** All multi-byte integers are little-endian. The only exception is CayenneLPP telemetry payload fields (big-endian). When in doubt, check the spec.

- **BLE MTU.** Request MTU 512 before sending any commands. If you skip this, frames larger than 20 bytes may fail silently on some OS/device combinations.

- **Timestamp units.** `CMD_SET_DEVICE_TIME` and outbound message timestamps use 32-bit Unix seconds (not milliseconds). `PACKET_MSG_SENT`'s suggested timeout is in **milliseconds**.

- **Message length.** Keep channel messages to 133 characters or fewer. Split longer messages at the application layer with chunk indicators (`[1/3] …`).

- **Concurrent flush protection.** If your framework can deliver `PUSH_CODE_MSG_WAITING` while a `get_msg()` loop is already running, use a lock or a debounce mechanism to prevent two concurrent drain loops.

- **One command at a time.** The command queue invariant is the single most common source of client bugs. If you ever have two pending commands, responses will be misrouted.

- **Contact list during drain.** The firmware may emit `PUSH_CODE_ADVERT` (`0x80`) while you are draining messages with `get_msg()`. Your receive loop must route adverts out of band — they are not a response to `get_msg()`.

- **Path length semantics differ by direction.** In `CMD_SEND_CHANNEL_DATA`, `path_len = 0xFF` means *flood*. In `RESP_CODE_CHANNEL_DATA_RECV`, `path_len = 0xFF` means *arrived via direct route*. The meaning is inverted. See the [Companion Protocol spec](https://docs.meshcore.io/companion_protocol/#receive-channel-data-datagram) for the full table.
