# Payload Types Tour

The 4-bit payload type field in the packet header identifies what the payload
contains and how every node on the mesh should treat it. This page walks
through each type — what it is for, who sends it, and what it triggers in a
receiver.

For the exact byte layout of each payload type, see the
[Payloads spec](https://docs.meshcore.io/payloads/) on `docs.meshcore.io`.

---

## Quick reference

| Value | Constant | Who sends | Encrypted? | Summary |
|-------|----------|-----------|------------|---------|
| `0x00` | `PAYLOAD_TYPE_REQ` | Companion, sensor | ✓ ECDH | Request to a specific known node |
| `0x01` | `PAYLOAD_TYPE_RESPONSE` | Repeater, room server, sensor | ✓ ECDH | Response to a REQ or ANON_REQ |
| `0x02` | `PAYLOAD_TYPE_TXT_MSG` | Companion | ✓ ECDH | Direct text message |
| `0x03` | `PAYLOAD_TYPE_ACK` | Any | ✗ | Delivery acknowledgment |
| `0x04` | `PAYLOAD_TYPE_ADVERT` | Any | ✗ (signed) | Node advertisement |
| `0x05` | `PAYLOAD_TYPE_GRP_TXT` | Companion | ✓ AES (shared) | Group channel text message |
| `0x06` | `PAYLOAD_TYPE_GRP_DATA` | Any | ✓ AES (shared) | Group channel binary datagram |
| `0x07` | `PAYLOAD_TYPE_ANON_REQ` | Companion | ✓ ECDH (ephemeral) | Request with inline public key |
| `0x08` | `PAYLOAD_TYPE_PATH` | Any | ✓ ECDH | Returned path for future direct routing |
| `0x09` | `PAYLOAD_TYPE_TRACE` | Companion, CLI | ✗ | SNR traceroute |
| `0x0A` | `PAYLOAD_TYPE_MULTIPART` | Any | — | Wrapper for multi-packet sequences |
| `0x0B` | `PAYLOAD_TYPE_CONTROL` | Any | ✗ | Discovery and control, zero-hop only |
| `0x0F` | `PAYLOAD_TYPE_RAW_CUSTOM` | Custom app | App-defined | Opaque bytes for custom applications |

---

## Types in depth

### `PAYLOAD_TYPE_TXT_MSG` — Direct text message

The workhorse of companion-to-companion communication. The payload wraps
an encrypted blob containing a 4-byte timestamp, a flags byte (plain text,
CLI command, or signed text), and the message string.

The encryption is ECDH-derived: only the sender and the intended recipient
can decrypt. Every repeater in the path relays the ciphertext opaquely.

Flooding is used on the first send (no path known). After a path is returned,
direct routing takes over, reducing airtime significantly.

---

### `PAYLOAD_TYPE_REQ` — Request to a known node

Used when a companion or sensor wants to invoke a function on a specific peer
— for example, getting stats from a repeater, sending a keepalive ping, or a
sensor request. The payload is ECDH-encrypted like a text message; the
application layer interprets the plaintext body.

Common request sub-types include `get stats` (battery, RSSI, packet counts),
`keepalive`, and application-specific requests such as requesting telemetry
from a sensor node.

---

### `PAYLOAD_TYPE_RESPONSE` — Response to REQ or ANON_REQ

The reply to a `REQ` or `ANON_REQ`. The payload structure mirrors `REQ` (same
wire envelope: dest hash, src hash, MAC, ciphertext) but the plaintext body is
application-defined. A repeater's stats response, for example, returns battery
millivolts, packet counters, and airtime totals; a room server's response
returns post count and membership state.

---

### `PAYLOAD_TYPE_ACK` — Acknowledgment

The simplest payload type: a 4-byte CRC checksum of the original message
(computed over its timestamp, text, and sender public key). Senders use this
to confirm delivery.

ACKs travel **directly** when the return path is known. They are also
propagated as floods if no path is available. The `multipart` ACK variant
(see `PAYLOAD_TYPE_MULTIPART` below) bundles multiple ACKs into one
transmission.

Because ACKs are unencrypted, any node can receive and forward them — the
CRC alone does not reveal message content.

---

### `PAYLOAD_TYPE_ADVERT` — Node advertisement

How nodes announce themselves. An advert payload contains:

- **Public key** (32 bytes) — the node's permanent Ed25519 identity.
- **Timestamp** (4 bytes) — when the advert was generated; used to detect
  stale entries.
- **Signature** (64 bytes) — Ed25519 signature over (public key + timestamp +
  app data), preventing forgery.
- **App data** — optional extension including the node type (Chat, Repeater,
  Room Server, Sensor), name string, and GPS lat/lon.

Adverts are **signed but not encrypted**. Any node that receives an advert can
validate it and add the sender to its contact list. See
[Adverts Deep Dive](adverts-deep-dive.md) for how they are constructed and
flooded.

---

### `PAYLOAD_TYPE_GRP_TXT` — Group channel text

A text message sent to a **channel** — a group identified by a shared secret
rather than by individual public keys. The payload uses the channel secret
(shared AES key) rather than ECDH, so any member who knows the channel secret
can decrypt.

The plaintext format is identical to `PAYLOAD_TYPE_TXT_MSG` inside, but the
wire envelope differs: instead of dest/src hashes, it carries a 1-byte
**channel hash** (first byte of SHA-256 of the channel secret). The receiver
scans its channel list for all channels whose hash matches, then tries to
decrypt with each.

!!! note "No per-sender authentication"
    Unlike direct messages, group text messages carry no per-sender
    cryptographic proof. Any node with the channel secret can claim any
    sender name. This is an intentional trade-off for simplicity.

---

### `PAYLOAD_TYPE_GRP_DATA` — Group channel binary datagram

The binary sibling of `GRP_TXT`. The payload carries an encrypted blob with
a 2-byte **data type** identifier (registered in the
[Number Allocations](https://docs.meshcore.io/number_allocations/) registry), a
length byte, and the application-specific data.

The data type registry allows multiple applications to coexist on the same
channel without interference. Custom applications should use the testing range
(`0xFF00–0xFFFF`) during development and request a permanent allocation before
publishing.

---

### `PAYLOAD_TYPE_ANON_REQ` — Anonymous request

Like `PAYLOAD_TYPE_REQ`, but the sender does not need to be in the
recipient's contact list. Instead of relying on a pre-cached shared secret,
the sender includes their **full 32-byte public key** in the payload, and the
recipient performs ECDH on the spot to derive the decryption key.

The primary use case is **first-time login** to a room server: the companion
sends an anonymous request containing the room password; the room server
derives the shared secret from the inline public key and validates the request,
then adds the sender to its member list so future messages can use ECDH
with the now-known public key.

---

### `PAYLOAD_TYPE_PATH` — Returned path

After a node receives a flood packet, it can send the **path** back to the
originator so that future messages can use direct routing. The payload
(ECDH-encrypted) contains:

- A `path_length` byte encoding the hash size and count.
- The ordered list of repeater hashes (the route from sender to receiver).
- An optional **extra** field: a bundled payload (e.g., an ACK or a response)
  piggybacked on the path return to save a round-trip.

The sender stores this path and uses it for all subsequent direct messages to
that peer, bypassing the need for a flood.

---

### `PAYLOAD_TYPE_TRACE` — SNR traceroute

A diagnostic packet that travels hop by hop along an **explicit path** (like a
direct packet), but instead of carrying a message, it collects the **SNR
reading** at each hop. When the final node in the path receives it, the path
field contains a sequence of SNR values (one signed byte per hop, encoded as
`SNR × 4`) that together describe the link quality along the route.

The companion app and CLI tools use TRACE to diagnose weak links and plan
repeater placement.

---

### `PAYLOAD_TYPE_MULTIPART` — Multi-packet sequence

A wrapper type that allows large payloads to be split across multiple radio
transmissions. The first byte of the payload encodes the number of remaining
packets in the sequence (upper 4 bits) and the inner payload type (lower 4
bits). Currently used for **multi-ACK** — bundling several ACK CRCs into one
frame to reduce airtime.

---

### `PAYLOAD_TYPE_CONTROL` — Discovery and control

Zero-hop control packets used for local-network discovery and coordination.
Currently includes:

- **DISCOVER_REQ** — broadcast by a node wanting to find neighbours of a
  specific type (chat node, repeater, room server, sensor).
- **DISCOVER_RESP** — a short reply carrying the node's type, SNR, and a
  tag reflecting the request.

Because these are zero-hop only (`getPathHashCount() == 0`), they are never
relayed by repeaters and never cross a mesh boundary.

---

### `PAYLOAD_TYPE_RAW_CUSTOM` — Custom application packet

An escape hatch for applications that need complete control over the payload
format and encryption. The firmware delivers raw bytes to
`Mesh::onRawDataRecv()` without attempting any decryption or interpretation.
Custom applications choose their own payload layout, their own cryptographic
scheme, and their own semantics.

Only `ROUTE_TYPE_DIRECT` is currently supported for raw custom packets —
they are not flooded.

---

## What's next

- [Routing and Flooding](routing-and-flooding.md) — how the route-type field
  drives packet propagation; how a returned path is built and used.
- [Encryption on the Wire](encryption-on-the-wire.md) — the ECDH and shared-key
  encryption models that most payload types rely on.
- [Adverts Deep Dive](adverts-deep-dive.md) — deep look at the advert payload
  and the flood mechanism that distributes it.
- [Payloads spec](https://docs.meshcore.io/payloads/) — byte-level field tables
  for every payload type listed above.
