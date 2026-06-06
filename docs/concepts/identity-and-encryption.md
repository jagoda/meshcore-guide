# Identity and Encryption

MeshCore is designed around a simple security principle: **cryptographic
identity, no central authority**. Every node generates its own key pair,
every direct message is end-to-end encrypted, and every advert is signed.
No account, no server, no registration.

This page explains the key model conceptually. For byte-level encoding of
signatures, MACs, and cipher blocks, see the
[Packet Format](https://docs.meshcore.nz/packet_format/) and
[Payloads](https://docs.meshcore.nz/payloads/) specs on `docs.meshcore.nz`.

---

## Key generation

When a MeshCore node boots for the first time it generates a fresh
**Ed25519 key pair**:

- **Private key (64 bytes):** stored on-device, never transmitted.
- **Public key (32 bytes):** broadcast in every advert and embedded in every
  direct-message packet. This is the node's permanent identity.

The private key can be backed up and restored via the CLI (`get prv.key` /
`set prv.key <hex>`) over a USB serial connection, but it must never leave the
device over RF.

!!! warning "Key hygiene"
    A node's identity is its key pair. If two devices share the same private
    key they are indistinguishable on the mesh. Keep private keys private; back
    them up to a secure offline location.

---

## Identity in code

From `src/Identity.h`:

```cpp
class Identity {
public:
  uint8_t pub_key[PUB_KEY_SIZE];   // 32 bytes (Ed25519 public key)

  bool verify(const uint8_t* sig, const uint8_t* message, int msg_len) const;
  // ...
};

class LocalIdentity : public Identity {
  uint8_t prv_key[PRV_KEY_SIZE];   // 64 bytes (Ed25519 private key)
public:
  void sign(uint8_t* sig, const uint8_t* message, int msg_len) const;
  void calcSharedSecret(uint8_t* secret, const Identity& other) const;
  // ...
};
```

`Identity` holds a *known peer*'s public key and can verify their signatures.
`LocalIdentity` is the node's own key pair and can sign and derive shared
secrets.

---

## Advert signing

Every advert packet is **signed with Ed25519** over the advert payload. A
receiver calls `Identity::verify()` on the sender's public key to confirm the
signature before storing the contact. A malicious node that replays or
fabricates an advert cannot forge the signature without the matching private
key.

---

## Direct message encryption (ECDH + AES-128)

Direct messages (`PAYLOAD_TYPE_TXT_MSG`) are encrypted end-to-end:

1. **Shared secret derivation.** The sender calls
   `LocalIdentity::calcSharedSecret()`, which performs an ECDH key exchange
   between the sender's private key and the recipient's public key (using
   Ed25519 keys transposed to X25519 / Curve25519 form). The result is a
   32-byte shared secret that only the two parties can derive.

2. **AES-128 encryption.** The payload (timestamp + message text) is encrypted
   with AES-128 using a 16-byte key derived from the shared secret.

3. **MAC.** A 2-byte message authentication code (MAC) is prepended. The
   receiver recomputes the MAC; a mismatch means the packet was tampered with
   or is addressed to the wrong node.

```
On wire: [dest hash][src hash][MAC (2 B)][AES-128 ciphertext]
```

Because the shared secret is symmetric (both sides derive the same value), the
recipient can decrypt without any additional key exchange round-trip.

!!! info "Key caching"
    Companion firmware pre-computes and caches the shared secret per contact
    so that each subsequent message does not need a fresh ECDH operation.

---

## Group channel encryption (AES-128, shared secret)

Group channel messages (`PAYLOAD_TYPE_GRP_TXT`) use a simpler model:

1. All members share a **channel secret** (a 16-byte key distributed
   out-of-band via QR code or secure copy).
2. The payload is AES-128 encrypted with a key derived from the channel
   secret.
3. A 2-byte MAC protects against corruption.

```
On wire: [channel hash (1–3 B)][MAC (2 B)][AES-128 ciphertext]
```

There is **no per-sender authentication** in a group message — any node that
knows the channel secret can send a believable message under any name. This is
an intentional trade-off for simplicity and low overhead on embedded hardware.

---

## The trust model

| Claim | How it is verified |
|-------|--------------------|
| "This advert is from node X" | Ed25519 signature over the advert payload |
| "This direct message is for me" | MAC check with my ECDH shared secret |
| "This channel message is from a member" | MAC check with the channel AES key |
| "Node X is who they say they are" | First-time: you accept the public key from their advert. Subsequent: signature. |

MeshCore does **not** implement a PKI or certificate authority. Trust is
**trust-on-first-use (TOFU)**: the first time you receive an advert from a
node, you add their public key to your contact list. After that, any packet
that fails the signature or MAC check is rejected. If you need to verify that a
public key truly belongs to a specific person, exchange it in person or via a
separate trusted channel.

---

## No central server required

The entire trust model works without internet connectivity:

- Key pairs are generated locally.
- Shared secrets are derived locally via ECDH.
- Signature verification is done locally against the stored public key.
- Channel secrets are distributed out-of-band (QR, radio, in-person).

A MeshCore mesh can operate in a completely air-gapped environment — no
internet, no DNS, no cloud service.

---

## What's next

- [Adverts and Contacts](adverts-and-contacts.md) — how public keys are
  distributed and stored.
- [Channels vs. Direct Messages](channels-vs-direct.md) — when to use shared
  channel secrets vs. per-contact ECDH.
- [Packet Format spec](https://docs.meshcore.nz/packet_format/) — the exact
  wire layout of MAC, IV, and ciphertext fields.
