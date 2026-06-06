# Common Operations

This page walks through the operations a typical Companion API client performs in the order it is likely to need them. Each section explains *why* the operation exists and *what to expect*, not the byte-level detail — for that, see the [Companion Protocol spec](https://docs.meshcore.nz/companion_protocol/) on docs.meshcore.nz and [Stats Binary Frames](https://docs.meshcore.nz/stats_binary_frames/) for stats commands.

## 1. Startup sequence

Perform these commands in order immediately after the transport connection is established. See [Command Lifecycle — Startup handshake](command-lifecycle.md#startup-handshake) for the full explanation.

```
CMD_APP_START          → PACKET_SELF_INFO
CMD_DEVICE_QUERY       → PACKET_DEVICE_INFO
CMD_SET_DEVICE_TIME    → PACKET_OK
CMD_GET_CONTACTS       → PACKET_CONTACT_START … PACKET_CONTACT(s) … PACKET_CONTACT_END
CMD_GET_CHANNEL × N    → PACKET_CHANNEL_INFO (one per slot)
CMD_SYNC_NEXT_MESSAGE  → (loop until PACKET_NO_MORE_MSGS)
```

Parse `PACKET_DEVICE_INFO` to learn `max_channels` — this tells you how many `GET_CHANNEL` calls to make. The firmware on older devices may report fewer channels; don't assume 8.

After the startup drain, set up your `PUSH_CODE_MSG_WAITING` handler so incoming messages are fetched promptly rather than waiting for your next periodic poll.

## 2. Fetching contacts

Contacts are fetched as a sequence: `CMD_GET_CONTACTS` triggers a start/item/end burst from the firmware:

```
App  → CMD_GET_CONTACTS
Node → PACKET_CONTACT_START   (signals list begins)
Node → PACKET_CONTACT          (one per contact, repeated)
Node → PACKET_CONTACT_END     (signals list complete)
```

Parse each `PACKET_CONTACT` immediately as it arrives; do not wait for `CONTACT_END` before processing. The contact payload includes the contact's public key (used as the stable identity across the mesh), advertised name, and routing information.

New contacts that appear on the mesh while your application is connected arrive as unsolicited `PUSH_CODE_ADVERT` (`0x80`) frames. If your application uses manual contact mode (`CMD_SET_MANUAL_ADD_CONTACTS`), these adverts are buffered for your application to decide whether to add them.

## 3. Sending a channel message

Channel messages are plain UTF-8 text broadcast to all nodes subscribed to a given channel slot. The firmware handles channel encryption transparently.

```python
import time, struct

CMD_SEND_CHANNEL_MSG = 0x03

def build_channel_msg(channel_idx: int, text: str, ts: int | None = None) -> bytes:
    ts = ts or int(time.time())
    encoded = text.encode("utf-8")
    return bytes([CMD_SEND_CHANNEL_MSG, 0x00, channel_idx]) \
         + struct.pack("<I", ts) \
         + encoded
```

The response is `PACKET_MSG_SENT` (`0x06`), which carries a **route flag** (direct vs. flood), a **tag** (expected ACK code), and a **suggested timeout** in milliseconds. Use the suggested timeout to decide how long to wait for an ACK before considering the transmission best-effort.

Keep text to **133 characters or fewer** per transmission. MeshCore's radio frame size limits effective message payload; messages longer than this must be split into numbered chunks by the application.

## 4. Receiving messages

Messages arrive in two ways:

1. **Proactively pushed** — the firmware sends `PUSH_CODE_MSG_WAITING` (`0x83`) when new messages are stored. Your application should call `CMD_SYNC_NEXT_MESSAGE` in a loop until `PACKET_NO_MORE_MSGS` (`0x0A`).
2. **Safety-net polling** — if your application suspects it missed a push code (e.g., after a reconnect), poll `CMD_SYNC_NEXT_MESSAGE` once to confirm the queue is empty.

The response to `CMD_SYNC_NEXT_MESSAGE` is one of:

| Response | Meaning |
|---|---|
| `PACKET_CHANNEL_MSG_RECV` (0x08) | Channel text message, standard format |
| `PACKET_CHANNEL_MSG_RECV_V3` (0x11) | Channel text message with SNR (firmware v3+) |
| `PACKET_CONTACT_MSG_RECV` (0x07) | Direct message, standard format |
| `PACKET_CONTACT_MSG_RECV_V3` (0x10) | Direct message with SNR (firmware v3+) |
| `PACKET_CHANNEL_DATA_RECV` (0x1B) | Binary datagram on a channel |
| `PACKET_NO_MORE_MSGS` (0x0A) | Queue empty — stop polling |

V3 formats (`0x10`, `0x11`) add an SNR byte (signed int8, scaled ×4 — divide by 4.0 for dB) before the standard fields. Always check the packet type byte before parsing to use the correct format.

Implement **message deduplication** using `(timestamp, pubkey_prefix)` or `(timestamp, channel_idx, text)` as a key. The firmware may deliver the same message more than once if the network receives multiple copies.

## 5. Channel management

Channels are identified by a slot index (0–7 on most firmware builds). Slot 0 is conventionally the public channel with the well-known public key. Slots 1–7 are available for private channels with app-generated secrets.

**Reading a channel slot:**

```
CMD_GET_CHANNEL [channel_idx]  →  PACKET_CHANNEL_INFO
```

`PACKET_CHANNEL_INFO` returns the channel name (32-byte UTF-8, null-padded) and the 16-byte shared secret. A slot is empty when both the name is empty and the secret is all zeros.

**Writing a channel slot:**

```
CMD_SET_CHANNEL [idx] [32-byte name] [16-byte secret]  →  PACKET_OK | PACKET_ERROR
```

Use a cryptographically secure random generator for the 16-byte secret; never use a hardcoded or weak secret for a private channel. To delete a slot, write an empty name and an all-zero secret.

The public channel secret is `8b3387e9c5cdea6ac9e5edbaa115cd72` (well-known; treat messages on this channel as public). Hashtag channels derive their secret from the first 16 bytes of `SHA-256("#channelname")`.

## 6. Reading device stats

The Companion API exposes three statistics sub-commands via `CMD_GET_STATS` (byte `0x38`, followed by a sub-type byte). These are **local queries** — they talk to the attached radio over the serial link and add no mesh traffic.

| Sub-type | Byte | Returns |
|---|---|---|
| `STATS_TYPE_CORE` | `0x00` | Battery millivolts, uptime seconds, error flags, queue length |
| `STATS_TYPE_RADIO` | `0x01` | Noise floor, last RSSI, last SNR (×4), TX/RX airtime seconds |
| `STATS_TYPE_PACKETS` | `0x02` | Cumulative recv/sent/flood/direct packet counters |

```python
CMD_GET_STATS   = 0x38
STATS_TYPE_CORE    = 0
STATS_TYPE_RADIO   = 1
STATS_TYPE_PACKETS = 2

# Get core stats
await send_and_wait(bytes([CMD_GET_STATS, STATS_TYPE_CORE]), expect=0x18)
```

The response byte is `RESP_CODE_STATS` (`0x18`), followed by the sub-type byte, then the typed payload. Byte-level layouts and parsing examples are in the [Stats Binary Frames spec](https://docs.meshcore.nz/stats_binary_frames/).

**SNR decoding reminder:** SNR values throughout the API are stored as signed int8 multiplied by 4. Divide by `4.0` to recover dB. For example, byte value `0xEC` (decimal 236, signed −20) → `−20 / 4.0 = −5.0 dB`.

## 7. Battery and storage

`CMD_GET_BATTERY` (`0x14`) returns `PACKET_BATTERY` (`0x0C`) with:

- Battery voltage in millivolts (uint16 LE) — divide by 1000.0 for volts.
- Used and total storage in kilobytes (uint32 LE each), when the firmware supports it.

Poll this periodically (not faster than once per minute) to monitor battery health without generating mesh traffic.
