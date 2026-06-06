# Design Constraints

Before you write a line of application code, understand the envelope. These
constraints are not bugs — they follow from LoRa physics and shared-spectrum
regulations. Designing with them in mind from the start saves painful
re-architectures later.

## MTU and fragmentation

The `MAX_PACKET_PAYLOAD` constant in `src/MeshCore.h` is **184 bytes**. That is
the total encrypted payload field. After MeshCore subtracts its own framing
overhead (hashes, MAC, timestamp, type byte), the bytes available for your
application data depend on the payload type you're using:

| Payload type | Overhead (approx.) | Usable application bytes |
|---|---|---|
| `PAYLOAD_TYPE_TXT_MSG` | ~22 bytes | ~162 bytes |
| `PAYLOAD_TYPE_REQ` / `RESPONSE` | ~22 bytes | ~162 bytes |
| `PAYLOAD_TYPE_GRP_DATA` | ~19 bytes + 2 type bytes | ~163 bytes |
| `PAYLOAD_TYPE_MULTIPART` | additional 4-byte header | variable |

Byte-level payload layouts are in the
[Payloads spec](https://docs.meshcore.io/payloads/) on docs.meshcore.io. Do
not re-derive them; look them up.

**If your message does not fit in a single packet**, MeshCore provides
`PAYLOAD_TYPE_MULTIPART` for splitting. Fragmentation adds complexity and
airtime — prefer designing messages that fit in one packet. In the worked
examples this means keeping payloads terse: a creature's state snapshot or an
escape-room clue should fit a single packet rather than span fragments. If you
must fragment, see the fragmentation pattern in
[Protocol Design Patterns](protocol-design-patterns.md#fragmentation-and-reassembly).

## Airtime and duty-cycle budget

Every transmission occupies the radio channel for the packet's **air time** —
calculated from spreading factor, bandwidth, coding rate, and payload length.
At SF10/BW250/CR5 (a typical configuration), a 184-byte payload takes roughly
500–700 ms on air.

Two limits constrain your transmit rate:

1. **MeshCore's `airtime_factor`** — a per-node throttle that limits how much
   of the radio channel a node consumes forwarding traffic. Your application
   traffic counts against this budget.
2. **Regional duty-cycle regulations** — in Europe (EU868) the legal limit is
   1 % duty cycle on most sub-bands, meaning a node can transmit for at most
   36 seconds per hour. The US (FCC Part 15) has no numeric duty-cycle limit
   but does limit dwell time and require frequency hopping on some bands.

Consult [Airtime and Regions](../concepts/airtime-and-regions.md) and
[Regions and Frequencies](../reference/regions-and-frequencies.md) for
region-specific numbers.

**Practical rule:** design your application's transmit rate so that, in the
busiest expected scenario, you consume no more than 10 % of the available
channel capacity. Leave room for adverts, ACKs, and other nodes.

## Latency and intermittent connectivity

MeshCore is **not a real-time network**. End-to-end latency for a multi-hop
direct message is:

```
latency ≈ Σ (airtime + retransmit_delay) for each hop
```

A 3-hop path with 700 ms air time per hop and a 2–4 second retransmit delay
window gives 8–14 seconds one-way. Flood routing adds additional variance.

Beyond per-packet latency, nodes are **intermittently connected**:

- Battery-powered nodes sleep between duty cycles.
- Nodes move in and out of radio range.
- Room servers go offline; repeaters reboot.

Design your application to work correctly when a target node is unreachable for
minutes, hours, or days. Store-and-forward via a room server is the standard
pattern for deferring delivery — see
[Protocol Design Patterns § Store-and-forward](protocol-design-patterns.md#store-and-forward).

## No delivery guarantee

MeshCore delivers best-effort. A `PAYLOAD_TYPE_ACK` tells you the immediate
next hop received the packet, not that the final destination did. `BaseChatMesh`
provides an end-to-end ACK mechanism (the `expected_ack_crc` pattern visible in
`examples/simple_secure_chat/main.cpp`), but:

- The ACK can be received more than once (multiple paths).
- The ACK can be lost even when the message arrived.
- Flood packets may arrive multiple times at the destination.

**You own reliability.** If your application requires confirmed delivery, you
must implement request/response with timeouts and retries at the application
layer. See
[Protocol Design Patterns § Acks, Retries, and Idempotency](protocol-design-patterns.md#acks-retries-and-idempotency).

## Addressing options

MeshCore offers two addressing models:

| Model | Mechanism | Use case |
|---|---|---|
| **Direct (unicast)** | 32-byte public key, resolved to 1-byte hash for routing | Point-to-point messages, REQ/RESPONSE |
| **Channel (group)** | Pre-shared symmetric key, hashed to channel ID | Broadcast to all nodes sharing the channel PSK |

Direct addressing requires you to know the recipient's public key. Nodes
discover each other via periodic **advertisements** (see
[Adverts and Contacts](../concepts/adverts-and-contacts.md) and
[Adverts Deep Dive](../protocol/adverts-deep-dive.md)).

Channel addressing reaches every node that knows the channel PSK. It is
unverified — anyone with the key can post. For trusted multicast, use
`PAYLOAD_TYPE_GRP_DATA` with your own application-layer signing, or use
`PAYLOAD_TYPE_ANON_REQ` for anonymous authenticated requests.

The [Number Allocations spec](https://docs.meshcore.io/number_allocations/)
on docs.meshcore.io assigns the type numbers for the `data_type` field in
`GRP_DATA` packets. If you invent a new data type, allocate a number there to
avoid collisions with other applications.

## The inherited security model

MeshCore encrypts unicast traffic with ECDH-derived shared secrets (Curve25519
+ AES-128-CTR + 2-byte MAC). Channel traffic uses a pre-shared AES key.

What this means for your application:

- **Unicast messages are confidential** between sender and recipient. An
  eavesdropper cannot read the payload.
- **Channel messages are confidential** from anyone who does not share the
  channel key, but are **unverified** — any node with the key can forge a
  channel message. Authenticate at the application layer if needed.
- **The 2-byte MAC** is a lightweight integrity check, not a cryptographic
  authentication tag. Replay protection is not guaranteed at the mesh layer —
  design idempotent handlers or add sequence numbers if replay is a concern.
- **Public keys are the identity.** There is no central CA. Contacts are
  trusted on first-advert (trust-on-first-use) unless your application adds
  an out-of-band verification step.

See [Encryption on the Wire](../protocol/encryption-on-the-wire.md) and
[Identity and Encryption](../concepts/identity-and-encryption.md) for the
full picture.

## Summary checklist

Before finalising your application protocol design, verify:

- [ ] Every message fits in 184 bytes, *or* you have a fragmentation plan.
- [ ] Transmit rate stays well inside duty-cycle limits under peak load.
- [ ] The application is correct when a node is unreachable for hours.
- [ ] Reliability is handled at the application layer (retries, idempotency).
- [ ] Addressing is clear: unicast by public key, or channel by PSK?
- [ ] Any security requirements beyond MeshCore's inherited model are
      implemented in your application layer.
