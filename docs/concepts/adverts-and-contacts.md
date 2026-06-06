# Adverts and Contacts

Before two MeshCore nodes can exchange messages, they need to know each other
exists. The **advert** mechanism is how nodes announce themselves, and the
**contact list** (address book) is where those announcements are stored.

---

## Your identity is your public key

Every MeshCore node generates an **Ed25519 key pair** the first time it boots.
The private key never leaves the device. The public key — 32 bytes — is the
node's permanent, unforgeable identity. There is no central registry, no account
to create, and no username/password to forget.

!!! info "Byte-level key format"
    For the exact wire encoding of a public key, see the
    [Payloads spec](https://docs.meshcore.nz/payloads/) on
    `docs.meshcore.nz`.

---

## What an advert contains

When a node advertises, it broadcasts a signed packet (`PAYLOAD_TYPE_ADVERT`)
that includes:

| Field        | Description |
|--------------|-------------|
| Public key   | The node's full 32-byte Ed25519 public key |
| Signature    | Ed25519 signature over the payload — proves the sender holds the matching private key |
| Timestamp    | Unix epoch seconds — used by receivers to detect stale adverts |
| Node type    | One of: Chat (Companion), Repeater, Room Server, Sensor |
| Name         | Human-readable label (optional but almost always set) |
| Lat / Lon    | GPS position (optional) |

The signature prevents spoofing: a malicious node cannot impersonate another
node without holding its private key.

---

## Two advertising modes

### Zero-hop advert

A zero-hop advert is broadcast once and **not relayed** by any repeater. Only
nodes that can hear the transmitter directly will receive it.

Use case: announcing yourself when you know you are in range of your contacts,
or when you want to avoid consuming channel airtime for a wide-area flood.

Companions advertise in zero-hop mode when you press the "advertise" button
without selecting flood mode.

### Flood advert

A flood advert is relayed by every repeater that receives it, propagating
across the entire reachable mesh. This is how nodes that are many hops apart
learn of each other.

**Repeaters** send a flood advert automatically every 12 hours (configurable
with `set flood.advert.interval <hours>`). **Companions** can send a flood
advert manually from the app.

---

## Path hashes: short IDs for routing

The full 32-byte public key is too long to embed in every packet header.
MeshCore uses a **path hash** — the first 1, 2, or 3 bytes of the public key —
as a compact node identifier inside packet paths.

| Hash size | Max hops | Notes |
|-----------|----------|-------|
| 1 byte    | 64       | Default; compatible with all firmware versions |
| 2 bytes   | 32       | Requires firmware ≥ 1.14 on all repeaters in the path |
| 3 bytes   | 21       | Better disambiguation for analysis tools; requires firmware ≥ 1.14 |

Because the first byte alone may collide across two repeaters with similar
public-key prefixes, the multibyte option was introduced in firmware 1.14.
Collisions do not break routing — packets still reach their destinations —
but they make path-analysis tools (traceroute, LetsMesh Analyser) harder to
interpret.

!!! info "Spec detail"
    The path hash size and count are encoded in the `path_len` field of the
    packet header. See the [Packet Format spec](https://docs.meshcore.nz/packet_format/)
    for the exact bit layout.

---

## Building your contact list

When your node receives an advert (zero-hop or flooded), it stores the sender's
public key, name, type, and last-seen timestamp in the **contact list**.

From this point on you can:

- Send a direct message to that contact (encrypted with the ECDH shared secret
  derived from your private key and their public key).
- See when they were last heard from.
- Share their QR code (contains public key + name + type) with someone who
  cannot hear their adverts directly.

!!! tip "QR and clipboard import"
    If a node is out of RF range, you can import a contact via QR code
    (`meshcore://contact/add?name=…&public_key=…&type=…`) or, on T-Deck,
    via a `clipboard.txt` file on the SD card.

---

## Why "last seen" can look stale

The "last seen" timestamp is set by the **clock on the receiving node**, not
the sending node. If either party has an incorrect clock (no GPS fix, wrong
time zone, or an unset T-Deck RTC), the displayed time will be wrong.
If contacts are not appearing in your list at all, the advert may have been
received before the node was in your list, or the node has not advertised
recently.

---

## What's next

- [How Messages Travel](how-messages-travel.md) — how path hashes are used to
  route packets hop by hop.
- [Identity and Encryption](identity-and-encryption.md) — the key model in
  depth: ECDH, AES, and MAC verification.
