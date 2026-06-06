# Encryption on the Wire

MeshCore protects message content so that repeaters in the middle of a path
can relay packets without reading them, and so that an eavesdropper with a
radio cannot recover plaintext. This page explains the key agreement model,
where encryption is applied, what is — and is not — encrypted in each packet
type, and why the design choices make sense for constrained embedded hardware.

For the byte-level encoding of keys, MACs, and ciphertext fields, see the
[Packet Format](https://docs.meshcore.io/packet_format/) and
[Payloads](https://docs.meshcore.io/payloads/) specs on `docs.meshcore.io`.

---

## Three protection models

MeshCore uses three different protection schemes depending on who the
intended audience is:

| Message type | Scheme | Who can decrypt |
|-------------|--------|----------------|
| Direct message / REQ / RESPONSE | **ECDH + AES-128** | Only sender and named recipient |
| Group channel text / data | **Shared AES-128** | Anyone with the channel secret |
| Advertisement | **Ed25519 signature only** (no encryption) | Everyone (signed, readable by all) |

---

## Keys: Ed25519, X25519, and derived secrets

Every MeshCore node holds an **Ed25519 key pair**:

- **Private key (64 bytes)** — stored on-device, never transmitted.
- **Public key (32 bytes)** — broadcast in adverts; the node's permanent identity.

Ed25519 is a signing curve. For encryption, MeshCore needs a key-*agreement*
curve. The firmware converts Ed25519 keys to **X25519 (Curve25519)** form
before performing ECDH. This conversion happens transparently inside
`LocalIdentity::calcSharedSecret()`.

The resulting **32-byte shared secret** is the same value whether computed by
the sender (`sender_prv × recipient_pub`) or the recipient
(`recipient_prv × sender_pub`). No key-agreement round-trip is needed — both
sides independently arrive at the same secret.

In code (`src/Identity.h`):

```cpp
// Called by the sender to derive the key for encrypting to 'other'
void LocalIdentity::calcSharedSecret(uint8_t* secret, const Identity& other) const;
```

!!! info "Key caching"
    Companion firmware pre-computes and caches the shared secret per contact
    so repeated messages do not incur ECDH computation overhead.

---

## Direct message encryption (ECDH + AES-128)

For `PAYLOAD_TYPE_TXT_MSG`, `PAYLOAD_TYPE_REQ`, `PAYLOAD_TYPE_RESPONSE`, and
`PAYLOAD_TYPE_PATH`, the encryption flow is:

1. **Derive the shared secret** — `LocalIdentity::calcSharedSecret()` performs
   ECDH between the sender's private key and the recipient's public key.

2. **Encrypt with AES-128** — the plaintext (timestamp + content) is encrypted
   with a 16-byte key derived from the shared secret. The symmetric cipher
   ensures both parties can encrypt and decrypt with the same key.

3. **Compute a 2-byte MAC** — a message authentication code is prepended to the
   ciphertext. On receive, `Utils::MACThenDecrypt()` verifies the MAC *before*
   decrypting. A MAC mismatch means the packet was corrupted, tampered with, or
   addressed to the wrong node (the MAC also binds the recipient identity).

The wire representation of the encrypted region:

```
[MAC (2 bytes)] [AES-128 ciphertext (remaining payload bytes)]
```

This is prefixed in the payload with the dest/src hashes:

```
payload: [dest_hash (1 B)] [src_hash (1 B)] [MAC (2 B)] [ciphertext]
```

---

## What repeaters see

A repeater receives the full packet. It can read:

- The **header byte** — necessary to classify the route type and payload type
  in order to decide how to forward.
- The **path field** — necessary to perform the "is this hash mine?" routing
  check.
- The **dest_hash** and **src_hash** prefix of the payload — these are
  unencrypted single-byte abbreviations of the public keys, needed for routing
  decisions.

A repeater **cannot** read:

- The message text.
- The timestamp.
- The request or response body.
- The returned path content (also ECDH-encrypted).

The 1-byte `dest_hash` leaks that a packet is *addressed to* a node with that
key prefix, but since one byte allows 256 possible values it provides only weak
traffic analysis data at best.

---

## Anonymous request encryption (ephemeral ECDH)

`PAYLOAD_TYPE_ANON_REQ` handles the case where the sender is not yet in the
recipient's contact list. Instead of relying on a pre-cached shared secret, the
sender includes their **full 32-byte public key** in the payload:

```
payload: [dest_hash (1 B)] [sender_pub_key (32 B)] [MAC (2 B)] [ciphertext]
```

The recipient performs ECDH on the spot using the inline public key and its own
private key to derive the shared secret, then decrypts. After a successful
anonymous request (e.g., room server login), the recipient adds the sender's
public key to its contact list so future messages can use the lighter
PAYLOAD_TYPE_REQ/TXT_MSG path.

---

## Group channel encryption (shared AES-128)

Group channels use a **pre-shared channel secret** rather than ECDH. All
members receive the same 32-byte secret out-of-band (via QR code or direct
copy).

The encryption flow is structurally the same — AES-128 with a 2-byte MAC — but
the key is derived from the shared channel secret rather than from an ECDH
exchange. The wire envelope replaces the dest/src hashes with a single
**channel hash** (first byte of SHA-256 of the channel secret):

```
payload: [channel_hash (1 B)] [MAC (2 B)] [ciphertext]
```

Any node that knows the channel secret can encrypt *and* decrypt group
messages. There is no per-sender authentication.

---

## Advertisement signatures (no encryption)

Advertisements (`PAYLOAD_TYPE_ADVERT`) are designed to be received and
processed by any node — including nodes that have never heard of the sender.
They cannot be encrypted with ECDH (no shared secret exists before the advert
is received).

Instead, each advert is **signed with Ed25519** over the combination of:

```
signed_message = [pub_key (32 B)] || [timestamp (4 B)] || [app_data (variable)]
```

The receiver calls `Identity::verify()` to check the signature before storing
the contact. A forged or replayed advert with a tampered public key cannot
produce a valid signature.

```cpp
// From Mesh::onRecvPacket(), PAYLOAD_TYPE_ADVERT case:
is_ok = id.verify(signature, message, msg_len);
if (is_ok) {
  onAdvertRecv(pkt, id, timestamp, app_data, app_data_len);
}
```

The timestamp in the signed payload means a replay of an old advert is still
valid (the signature covers the original timestamp), but receivers that already
hold a newer-timestamped advert for that public key can detect and ignore stale
replays.

---

## What is NOT protected

The following are **plaintext on the wire** in all current packet types:

| Field | Reason |
|-------|--------|
| Header byte | Needed by every node to classify and route the packet |
| Path field | Needed by each repeater to decide whether to forward |
| `dest_hash` / `src_hash` (1 byte each) | Needed for routing lookups; only 1-byte prefix leaks |
| ACK CRC | Intentional: ACKs are meant to be relayed freely; no sensitive data |
| Advert payload | Intended to be publicly readable; protected by signature, not encryption |
| Control / discovery packets | Designed for local broadcast; no sensitive data |

---

## Security properties in summary

| Property | How achieved |
|----------|-------------|
| Message confidentiality (direct) | AES-128 with ECDH-derived key; intermediate nodes cannot decrypt |
| Message integrity | 2-byte MAC; failed MAC → packet silently dropped |
| Sender authentication (direct) | MAC binds to the sender's ECDH secret; only the holder of the private key could have produced it |
| Identity authenticity (adverts) | Ed25519 signature; cannot be forged without the private key |
| Group membership privacy | Channel secret distributed out-of-band; only members can decrypt |
| No central authority | All keys generated locally; no certificate infrastructure required |

---

## What's next

- [Adverts Deep Dive](adverts-deep-dive.md) — how Ed25519 signatures are
  applied and verified in the advert flow.
- [Routing and Flooding](routing-and-flooding.md) — what repeaters can and
  cannot see as they forward packets.
- [Payloads spec](https://docs.meshcore.io/payloads/) — byte-level layout of
  the MAC, ciphertext, and key fields for each payload type.
