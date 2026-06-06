# Adverts Deep Dive

An advertisement is how a MeshCore node announces its existence to the mesh.
Without adverts, nodes have no way to discover each other's public keys, types,
or names. This page explains how adverts are constructed and signed, how they
are sent (zero-hop vs. flood), what triggers a re-advert, and how a receiving
node processes and stores them.

Byte-level field tables are in the [Payloads spec](https://docs.meshcore.io/payloads/)
and [Number Allocations](https://docs.meshcore.io/number_allocations/) on
`docs.meshcore.io`.

---

## What an advert carries

An advert packet (`PAYLOAD_TYPE_ADVERT`) has three mandatory fields followed
by optional **app data**:

| Field | Size | Content |
|-------|------|---------|
| Public key | 32 bytes | Sender's Ed25519 public key — the node's permanent identity |
| Timestamp | 4 bytes | Unix epoch seconds at advert generation |
| Signature | 64 bytes | Ed25519 signature over `[pub_key ‖ timestamp ‖ app_data]` |
| App data | variable | Optional; see below |

The signature covers both the fixed fields *and* the app data. Changing any
part of the advert — even a single bit in the name — invalidates the signature.
A malicious node cannot impersonate another without the matching private key.

---

## App data: the advert's personality

App data is an optional byte sequence that carries node-specific metadata. Its
format is defined in `src/helpers/AdvertDataHelpers.h`:

```cpp
#define ADV_TYPE_NONE     0
#define ADV_TYPE_CHAT     1   // Companion / chat node
#define ADV_TYPE_REPEATER 2
#define ADV_TYPE_ROOM     3   // Room server
#define ADV_TYPE_SENSOR   4

#define ADV_LATLON_MASK   0x10  // lat/lon present
#define ADV_FEAT1_MASK    0x20  // reserved
#define ADV_FEAT2_MASK    0x40  // reserved
#define ADV_NAME_MASK     0x80  // name string present
```

The first byte of app data is a **flags byte**: the lower 4 bits encode the
node type, and the upper 4 bits are a presence bitmap for optional fields.

The optional fields follow in this order when their flag bit is set:

| Field | Size | Flag bit |
|-------|------|----------|
| Latitude | 4 bytes (int32, decimal × 10⁶) | `ADV_LATLON_MASK` |
| Longitude | 4 bytes (int32, decimal × 10⁶) | `ADV_LATLON_MASK` |
| Feature 1 | 2 bytes | `ADV_FEAT1_MASK` (reserved) |
| Feature 2 | 2 bytes | `ADV_FEAT2_MASK` (reserved) |
| Name string | remainder | `ADV_NAME_MASK` |

`AdvertDataBuilder::encodeTo()` writes this sequence; `AdvertDataParser` reads
it back. Both are straightforward serialisers; the firmware uses them rather
than writing ad hoc byte manipulation.

!!! example "Encoding a repeater with a name and location"
    A repeater named "Summit R1" at 37.422°N, 122.084°W would produce an
    app-data byte sequence of:

    ```
    flags = 0x02 | 0x10 | 0x80  → node type REPEATER + HAS_LATLON + HAS_NAME
    [flags][lat (4 B)][lon (4 B)][Summit R1]
    ```

---

## Two send modes

### Zero-hop advert

A zero-hop advert is sent with `sendZeroHop()`. The path field has zero hops
and the route type is `ROUTE_TYPE_DIRECT`. Repeaters that receive it will *not*
relay it — the advert is heard only by directly reachable nodes.

**When to use:** when the sender knows its contacts are within one hop, or
when it wants to minimise airtime consumption.

Companions typically send zero-hop adverts when the user presses "advertise"
manually in the app without choosing the flood option.

### Flood advert

A flood advert is sent with `sendFlood()` using `ROUTE_TYPE_FLOOD`. Repeaters
relay it as they would any other flood packet, appending their hashes and
propagating the advert across the entire reachable mesh.

**When to use:** when the node needs to be discoverable by nodes many hops away,
or when a repeater needs to announce its presence at startup and on its
scheduled interval.

!!! info "Automatic flood advert scheduling"
    Repeaters send a flood advert automatically at startup and every 12 hours
    by default (configurable with `set flood.advert.interval <hours>` via CLI).
    This ensures that new nodes joining the mesh can discover existing
    repeaters even if they weren't present during the last manual advert.

---

## What triggers a re-advert

Several events cause a node to re-send its advert:

| Trigger | Mode | Why |
|---------|------|-----|
| Firmware boot | Flood | Announces presence to the mesh at startup |
| Scheduled interval (repeaters) | Flood | Keeps the mesh contact list fresh |
| User presses "advertise" in app | Zero-hop or flood (user choice) | Manual re-announcement |
| Location update (GPS fix) | Flood | Updates lat/lon in the advert |
| Name or configuration change | Flood | Ensures contacts see updated metadata |

The timestamp in each advert is the wall-clock time at generation. Receiving
nodes store the newest-timestamped advert per public key and discard older ones.

---

## Receiving and validating an advert

When a node receives a `PAYLOAD_TYPE_ADVERT` packet, the processing in
`Mesh::onRecvPacket()` goes through four checks before the advert is acted on:

1. **Complete packet** — verify that the packet is long enough to contain all
   mandatory fields (pub_key + timestamp + signature).

2. **Not self** — if the public key matches the node's own `self_id`, discard.
   A node should not process its own flooded advert.

3. **Not a duplicate** — check `_tables->hasSeen(pkt)`. If the same advert has
   already been processed (e.g., via a different path), skip processing but
   still relay.

4. **Signature verification** — reconstruct the signed message
   `[pub_key ‖ timestamp ‖ app_data]` and call `Identity::verify(signature, ...)`.
   If the signature is invalid, the advert is silently dropped — no relay, no
   contact storage.

```cpp
// From Mesh::onRecvPacket(), PAYLOAD_TYPE_ADVERT case (condensed):
is_ok = id.verify(signature, message, msg_len);
if (is_ok) {
    onAdvertRecv(pkt, id, timestamp, app_data, app_data_len);
    action = routeRecvPacket(pkt);   // relay the advert further
}
```

Only after signature verification does `onAdvertRecv()` fire — allowing the
application layer (companion firmware) to update its contact list. Relaying
also happens only on successful verification.

---

## How the path accumulates during a flood advert

Because adverts use flood routing, each relaying repeater appends its own hash
to the path field (as described in [Routing and Flooding](routing-and-flooding.md)).
By the time the advert reaches a distant node, its path field contains the
ordered hashes of every repeater it traversed.

This path is not stored as a "route to the advert sender" by default — it is
the route the *advert* took, which is the reverse of a route back to that node.
Some companion firmware implementations use the incoming advert path to
bootstrap a direct route, but this is application-layer behaviour, not a
protocol requirement.

---

## Advert replay and freshness

Because the signature covers the timestamp, a replayed advert (one captured
and re-transmitted by a malicious node) is still cryptographically valid.
The timestamp is the freshness signal:

- Nodes that receive an advert with a timestamp older than their stored entry
  for that public key can discard it as stale.
- The `seen-packet` table (`SimpleMeshTables`) also suppresses relay of
  recently-seen copies of the same advert, limiting the blast radius of any
  replay attempt.

No absolute expiry mechanism exists in the current protocol — a sufficiently
old advert can still be injected and propagated, though it would appear stale
to any node comparing timestamps.

---

## What's next

Within The Protocol section:

- [Routing and Flooding](routing-and-flooding.md) — how the flood mechanism
  that carries adverts works in detail.
- [Encryption on the Wire](encryption-on-the-wire.md) — why adverts are signed
  but not encrypted, and how the signature protects them.
- [Payloads spec](https://docs.meshcore.io/payloads/) — byte-level advert
  field layout including appdata flags and optional field encoding.
- [Number Allocations](https://docs.meshcore.io/number_allocations/) — registry
  for group data-type identifiers used alongside group adverts.

**Continuing the arc:** You have completed The Protocol section. The next section
is [Companion API →](../companion-api/index.md), which shows how external
applications — Home Assistant integrations, custom tools, mobile apps — speak to
a Companion Radio node using the binary frame protocol that sits on top of
everything you just learned.
