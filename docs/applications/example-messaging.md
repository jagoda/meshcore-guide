# Worked Example: Email-like Messaging

**Patterns used:**
[store-and-forward](protocol-design-patterns.md#store-and-forward),
[request/response correlation](protocol-design-patterns.md#requestresponse-correlation),
[acks/retries/idempotency](protocol-design-patterns.md#acks-retries-and-idempotency),
[protocol versioning](protocol-design-patterns.md#protocol-versioning)

**Primary code anchor:** `examples/simple_secure_chat` (firmware-side chat
node — the closest existing analogue, though this example is host-side via the
Companion API).

## What we're building

A store-and-forward messaging application with the following properties:

- Messages are addressed to a recipient by public key (inbox model).
- A **room server** acts as the mail host: it buffers messages for offline
  recipients and pushes them when the recipient reconnects.
- Senders get a **delivery receipt** when the server accepts the message and
  a separate **read receipt** (optional) when the recipient reads it.
- Threads are identified by a subject hash; replies reference the parent.
- The application runs **host-side** via the Companion API — no firmware
  changes required.

## Architecture

```
Sender (host app)
    │ Companion API (serial/BLE)
    ▼
Companion Radio node ──[LoRa]──► Room Server node ──[LoRa]──► Recipient node
                                   (mail host)                  │ Companion API
                                                               Recipient host app
```

The Companion Radio node is a radio modem for the host app. The room server
is a standard `simple_room_server` firmware build, used as a relay. The only
custom code is on the two host applications (sender and recipient).

## Message format

Messages travel as `PAYLOAD_TYPE_GRP_DATA` packets on a private channel
shared by all users of the application. The `data_type` field (a 16-bit
integer in the GRP_DATA payload — see
[Payloads spec](https://docs.meshcore.nz/payloads/)) is allocated for this
application from the [Number Allocations spec](https://docs.meshcore.nz/number_allocations/).

Envelope format (application bytes inside the GRP_DATA blob):

```
┌─────────┬──────┬──────────────────┬────────────────────┬──────────────────────┐
│ version │ type │  msg_id (4 B)    │ recipient_key (7 B) │ application payload  │
│  1 B    │  1 B │                  │ (pub key prefix)    │ (variable)           │
└─────────┴──────┴──────────────────┴────────────────────┴──────────────────────┘
```

The 7-byte public key prefix is enough to route within the room server's
contact table without spending 32 bytes on the full key.

### Message types

| Type byte | Name | Direction | Payload |
|---|---|---|---|
| `0x01` | `MSG_SEND` | sender → server | subject hash (4 B), body (up to ~130 B) |
| `0x02` | `MSG_ACK` | server → sender | `msg_id` echo, status byte |
| `0x03` | `MSG_DELIVER` | server → recipient | `msg_id`, sender key prefix (7 B), subject hash (4 B), body |
| `0x04` | `MSG_READ_ACK` | recipient → server | `msg_id` echo |
| `0x05` | `MSG_READ_NOTIFY` | server → sender | `msg_id`, read timestamp |

### Keeping it within MTU

At SF10/BW250 the usable GRP_DATA payload is ~163 bytes. The envelope overhead
is 1 + 1 + 4 + 7 = 13 bytes. For `MSG_SEND`, that leaves:
4 (subject hash) + up to 146 bytes of body. Constraining body to 140 bytes
leaves 6 bytes of headroom. If longer bodies are needed, use the
[fragmentation pattern](protocol-design-patterns.md#fragmentation-and-reassembly)
with a `msg_id`-keyed reassembly buffer on the server.

## Subject and threading

A **subject hash** is the first 4 bytes of SHA-256 of the canonical subject
string (lowercased, stripped). Replies set the same subject hash as the
original message, creating a thread. The host application is responsible for
displaying threads; the mesh layer is agnostic to the hash.

4 bytes gives 2³² possible subject hashes, which is collision-resistant
enough for a small group. If your user base is large enough to worry about
collisions, expand to 6 bytes and document the version bump.

## Offline delivery via room server

The room server buffers `MSG_DELIVER` frames keyed by `(recipient_key_prefix,
msg_id)`. When the recipient Companion Radio node connects to the server (via
its periodic advert cycle), the server pushes queued messages as GRP_DATA
packets on the shared channel.

Design choices and their pattern references:

| Choice | Pattern |
|---|---|
| `msg_id` is a random 32-bit nonce from sender | [idempotency](protocol-design-patterns.md#acks-retries-and-idempotency) — server deduplicates on re-delivery |
| Server sends `MSG_ACK` when it accepts a `MSG_SEND` | [request/response correlation](protocol-design-patterns.md#requestresponse-correlation) |
| Sender retries `MSG_SEND` with same `msg_id` if no `MSG_ACK` in 60 s | [retries](protocol-design-patterns.md#acks-retries-and-idempotency) |
| Server stores messages with a 7-day TTL | [store-and-forward](protocol-design-patterns.md#store-and-forward) |
| Server limits queue to 32 messages per recipient | [store-and-forward](protocol-design-patterns.md#store-and-forward) |

## Delivery receipts

The `MSG_ACK` from the server to the sender signals *server acceptance*, not
recipient delivery. For true delivery receipts:

1. Server sends `MSG_DELIVER` to recipient.
2. Recipient host app replies with `MSG_READ_ACK` (when user opens the
   message, not just on receipt).
3. Server forwards a `MSG_READ_NOTIFY` to the original sender.

Read receipts are **best-effort**: if the sender is offline when the
`MSG_READ_NOTIFY` arrives, it is lost. The server does not store read receipts.
Document this limitation in your app's UX.

## Addressing by public key

The recipient's full 32-byte public key is known from prior advert exchange
(see [Adverts and Contacts](../concepts/adverts-and-contacts.md)). The host
app passes the 7-byte prefix in the message envelope. The room server resolves
the prefix to a full contact entry in its table. If resolution is ambiguous
(two contacts share a 7-byte prefix — vanishingly rare but possible), the
server returns an error status in `MSG_ACK` and the sender should use a longer
prefix or the full key.

## Implementation surface decision

This example runs **host-side** via the Companion API because:

- All logic (threading, receipt tracking, retry timers) can live on a capable
  host machine.
- The Companion Radio node acts purely as a radio modem.
- The room server is stock firmware; no firmware changes needed.

If you needed the relay logic to run without a host computer (e.g., a
solar-powered mail relay in the field with no Pi attached), you would need a
custom firmware build per [Custom Application](../extending/custom-application.md).

## Cross-links

- [Companion API](../companion-api/index.md) — how to send GRP_DATA frames
  from a host application.
- [Running a Room Server](../operating/running-a-room-server.md) — operating
  the relay node.
- [Payload Types Tour](../protocol/payload-types-tour.md) — GRP_DATA in context.
- [Protocol Design Patterns](protocol-design-patterns.md) — all patterns
  referenced above.
- [Building and Testing](building-and-testing.md) — dev loop for this
  example without live RF.
