# Protocol Design Patterns

These patterns are the shared vocabulary for the three worked examples. Pick
the ones your application needs; skip the rest. Each pattern is described once
here; the examples reference it by name rather than repeat the explanation.

## Message framing and type bytes

Every application payload should start with a **type byte** (or short type
field) that identifies the message kind. This lets the receiver dispatch to the
right handler without guessing from context.

```
┌──────┬──────────────────────────────────┐
│ type │ application payload              │
│  1 B │ up to ~161 B                     │
└──────┴──────────────────────────────────┘
```

Conventions:

- Reserve `0x00` as "unknown/unrecognised" so that a zero-initialised buffer
  is never a valid message.
- Group types by direction if relevant: `0x01–0x7F` = client→device,
  `0x80–0xFF` = device→client (this is the pattern the Companion API uses —
  see [Frame Model](../companion-api/frame-model.md)).
- If you introduce a new `GRP_DATA` data type, register a number in the
  [Number Allocations spec](https://docs.meshcore.io/number_allocations/).

For host-side applications using the Companion API, your type byte lives
inside the blob you pass to the send-message or send-data frame. The mesh
layer does not inspect it. For firmware-side applications, register a
`PAYLOAD_TYPE_*` constant in the range `0x0C–0x0E` (currently unassigned;
check `src/Packet.h` and the spec before choosing a value).

## Request/response correlation

The mesh is **async**: you send a request and, some seconds later (or never),
a response arrives. Correlate them with a **request ID**.

```
Request:   [type=REQ][req_id: 2B][payload...]
Response:  [type=RESP][req_id: 2B][status: 1B][payload...]
```

Use a monotonically increasing 16-bit counter, or a random 16-bit nonce, as
the request ID. The responder echoes it unchanged. Your application keeps a
table of outstanding requests keyed by ID; on response, look up the entry,
call the completion handler, and remove it.

For `PAYLOAD_TYPE_REQ` / `PAYLOAD_TYPE_RESPONSE` pairs, MeshCore already
manages the encryption and routing; your request ID travels inside the
encrypted payload blob, invisible to the mesh layer. See
[Companion API § Command Lifecycle](../companion-api/command-lifecycle.md) for
how the Companion Radio firmware implements this pattern at the frame level.

## Acks, retries, and idempotency {#acks-retries-and-idempotency}

MeshCore's mesh-layer ACK (`PAYLOAD_TYPE_ACK`) confirms reception at the next
hop, not end-to-end. For confirmed delivery:

1. **Send the request.** Start a retry timer (e.g., 30 s).
2. **Wait for an application-layer response** (your own response message that
   echoes the `req_id`).
3. **If no response before timeout**, retransmit with the same `req_id` (so
   the receiver can detect a duplicate) and double the timer (exponential
   backoff).
4. **Give up after N retries** and surface an error to the user.

**Idempotency:** because duplicates can arrive (from multiple paths *and* from
your retries), handlers must be safe to call more than once with the same
`req_id`. Pattern: store the last-seen `req_id` per sender; if you see a
duplicate, re-send your cached response without re-executing the action.

The `expected_ack_crc` mechanism in `examples/simple_secure_chat/main.cpp` is
a lightweight single-outstanding-request version of this pattern.

## Fragmentation and reassembly {#fragmentation-and-reassembly}

If a single logical message exceeds ~160 bytes of application payload, you must
split it across multiple packets. MeshCore provides `PAYLOAD_TYPE_MULTIPART`
for this, but you can also implement fragmentation in your application layer
over `GRP_DATA` or REQ/RESPONSE.

Application-layer fragmentation header (example):

```
┌──────────┬───────┬───────┬──────────────────┐
│ msg_id   │ frag  │ total │ fragment payload  │
│  2 B     │  1 B  │  1 B  │ up to ~157 B      │
└──────────┴───────┴───────┴──────────────────┘
```

- `msg_id`: unique per logical message (from sender).
- `frag`: 0-based fragment index.
- `total`: total fragment count.

Reassembly: buffer fragments keyed by `(sender_id, msg_id)`. When all
fragments arrive, reassemble and deliver. Expire incomplete assemblies after a
timeout (e.g., 5 minutes).

**Design note:** fragmentation multiplies airtime consumption by the fragment
count *and* multiplies loss probability. A 3-fragment message at 1 % packet
loss per hop has roughly 3 % loss probability before retries. Strongly prefer
fitting in a single packet.

## Sequencing and ordering

Packets may arrive out-of-order (different flood paths, retransmits). If your
application needs ordered delivery, add a **sequence number** to each message
and buffer out-of-order arrivals.

```
┌──────┬──────────┬──────────────────────┐
│ type │  seq_no  │ payload              │
│  1 B │   2 B    │ ...                  │
└──────┴──────────┴──────────────────────┘
```

Maintain per-sender expected-next-sequence. Deliver in-order; hold
out-of-order for a configurable window; drop if the window fills (prefer
liveness over ordering guarantees when the channel is lossy).

For most mesh applications, ordering is less important than delivery — prefer
idempotent unordered handlers and reconcile state at the application level.

## Store-and-forward {#store-and-forward}

When the destination node is offline, you need a **relay** that holds the
message until the destination reconnects. MeshCore's room servers (`simple_room_server`)
implement this natively for text messages. For application data:

**Option A — Room server as mailbox.** Send your application message as a
`PAYLOAD_TYPE_GRP_DATA` to a channel the room server is subscribed to. The room
server stores it and pushes it to connected clients. This works without any
firmware changes and is the pattern used in the messaging example.

**Option B — Custom firmware relay.** Add a firmware-side store-and-forward
buffer using `PAYLOAD_TYPE_REQ` / `RESPONSE`. The relay node accepts requests
on behalf of offline destinations, stores them, and forwards when the
destination advertises. This requires a custom firmware build (see
[Extending MeshCore](../extending/index.md)).

In both cases:
- Tag stored messages with a TTL. Expire and discard after TTL elapses.
- Limit queue depth per destination to bound memory use.
- Deduplicate on `req_id` or content hash to handle retransmissions from the sender.

## Discovery and announce

Nodes advertise their presence periodically via `PAYLOAD_TYPE_ADVERT`. Your
application can:

1. **Piggyback on adverts** — include a short app-type identifier in the advert
   data field (see [Adverts Deep Dive](../protocol/adverts-deep-dive.md)). Other
   nodes see your type in `contact.type` (ADV_TYPE_* constants) and know you
   support your application protocol.
2. **Send a dedicated discovery packet** — broadcast a `PAYLOAD_TYPE_GRP_DATA`
   with a "hello, I support app X, version Y" type byte to a well-known channel.
   Nodes that support the app reply with their public key.

For most applications, option 1 is sufficient. The room server pattern (option
A above) means any node can post without pre-discovering the server's public key
if the channel PSK is shared out-of-band.

## Rate limiting and backoff

The mesh is a shared channel. Do not burst-transmit. Rules of thumb:

- **Minimum inter-message gap:** at least 2× the estimated round-trip air time
  (2 × hop_count × airtime_per_packet).
- **Exponential backoff on retry:** start at 30 s, double each retry, cap at
  10 minutes.
- **Respect `airtime_factor`:** if you are using the Companion API, the node
  will queue or drop packets that exceed its budget. Build a local queue with
  backpressure rather than hammering the serial interface.

The IoT control example shows a practical duty-cycle-aware polling pattern.

## Protocol versioning

Include a **version byte** in your application protocol framing. Send it in
every message.

```
┌─────────┬──────┬──────────────────────────┐
│ version │ type │ payload                  │
│  1 B    │  1 B │ ...                      │
└─────────┴──────┴──────────────────────────┘
```

- On receipt, check the version. If it is higher than your max-supported
  version, respond with an error type and your own current version.
- On receipt, if it is lower than your min-supported version, respond with an
  error or ignore (your choice; document the policy).
- Never change the meaning of existing fields without bumping the version.
- For channel messages (unverified), be tolerant of unknown fields in newer
  versions — parse what you know, ignore the rest.

This is especially important for firmware-side payload types, where rolling
updates across a distributed mesh mean nodes will temporarily be at different
versions.
