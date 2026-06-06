# Packet & Identity

These two pairs of classes are the data layer of MeshCore:

- **`Packet`** (`src/Packet.h`, `src/Packet.cpp`) — the fundamental
  transmission unit; everything that goes over the radio is a `Packet`.
- **`Identity` / `LocalIdentity`** (`src/Identity.h`, `src/Identity.cpp`) —
  Ed25519 key pairs that identify every node.

## Packet

### What a `Packet` is

```cpp
class Packet {
public:
  uint8_t  header;                        // route type + payload type + version (1 byte)
  uint16_t payload_len, path_len;         // lengths of the two variable fields
  uint16_t transport_codes[2];            // optional transport / region scope codes
  uint8_t  path[MAX_PATH_SIZE];           // flood path (up to 64 bytes of hashes)
  uint8_t  payload[MAX_PACKET_PAYLOAD];   // encrypted payload (up to 184 bytes)
  int8_t   _snr;                          // SNR of received packet (×4, signed)
};
```

`MAX_PATH_SIZE` is 64 bytes; `MAX_PACKET_PAYLOAD` is 184 bytes.
The wire encoding is compact — see the spec for exact byte layout.

### The header byte

The single `header` byte packs three fields:

```
bits [1:0]  route type   (ROUTE_TYPE_FLOOD, DIRECT, TRANSPORT_FLOOD, TRANSPORT_DIRECT)
bits [5:2]  payload type (PAYLOAD_TYPE_TXT_MSG, ADVERT, ACK, …)
bits [7:6]  payload ver  (currently always PAYLOAD_VER_1 = 0x00)
```

Accessor methods handle the bit-field extraction:

```cpp
uint8_t getRouteType()   const { return header & PH_ROUTE_MASK; }
uint8_t getPayloadType() const { return (header >> PH_TYPE_SHIFT) & PH_TYPE_MASK; }
uint8_t getPayloadVer()  const { return (header >> PH_VER_SHIFT) & PH_VER_MASK; }
```

### Route types

| Constant | Value | Meaning |
|---|---|---|
| `ROUTE_TYPE_TRANSPORT_FLOOD` | `0x00` | Flood + transport codes attached |
| `ROUTE_TYPE_FLOOD` | `0x01` | Flood; path built up hop-by-hop |
| `ROUTE_TYPE_DIRECT` | `0x02` | Direct; full path supplied at send |
| `ROUTE_TYPE_TRANSPORT_DIRECT` | `0x03` | Direct + transport codes |

### Payload types

| Constant | Value | Description |
|---|---|---|
| `PAYLOAD_TYPE_REQ` | `0x00` | Encrypted request to a known peer |
| `PAYLOAD_TYPE_RESPONSE` | `0x01` | Encrypted response |
| `PAYLOAD_TYPE_TXT_MSG` | `0x02` | Encrypted text message |
| `PAYLOAD_TYPE_ACK` | `0x03` | Simple acknowledgement (CRC of acked packet) |
| `PAYLOAD_TYPE_ADVERT` | `0x04` | Node advertisement (public key, name, capabilities) |
| `PAYLOAD_TYPE_GRP_TXT` | `0x05` | Encrypted group text message |
| `PAYLOAD_TYPE_GRP_DATA` | `0x06` | Encrypted group datagram |
| `PAYLOAD_TYPE_ANON_REQ` | `0x07` | Request with ephemeral public key (anonymous sender) |
| `PAYLOAD_TYPE_PATH` | `0x08` | Returned routed path (enables direct messaging) |
| `PAYLOAD_TYPE_TRACE` | `0x09` | Trace-route packet (collects per-hop SNR) |
| `PAYLOAD_TYPE_MULTIPART` | `0x0A` | One segment of a multi-part packet |
| `PAYLOAD_TYPE_CONTROL` | `0x0B` | Control / node-discovery packet |
| `PAYLOAD_TYPE_RAW_CUSTOM` | `0x0F` | Raw bytes; custom encryption / payload |

> **Cross-link:** For the full byte-level encoding of each payload type →
> [`docs.meshcore.io/payloads`](https://docs.meshcore.io/payloads).

### Path encoding

`path_len` is not simply a byte count — it is an encoded field:

```
bits [7:6]  path_hash_size − 1   (0 → 1-byte hashes, 1 → 2-byte, …)
bits [5:0]  path_hash_count      (number of hashes in path[])
```

Accessor methods:

```cpp
uint8_t getPathHashSize()  const;  // 1..4 bytes per hash
uint8_t getPathHashCount() const;  // number of hashes
uint8_t getPathByteLen()   const;  // getPathHashCount() × getPathHashSize()
```

For standard V1 packets `PATH_HASH_SIZE = 1`, so a path of 3 hops occupies
3 bytes. The hash of each intermediate node is just the first byte of that
node's 32-byte public key. This is why the `pub_key[0]` values `0x00` and
`0xFF` are reserved — they would be ambiguous in a path.

### Wire serialisation

```cpp
int  getRawLength() const;                     // encoded byte count
uint8_t writeTo(uint8_t dest[]) const;         // serialise to buffer
bool    readFrom(const uint8_t src[], uint8_t len); // deserialise from buffer
```

`tryParsePacket()` in `Dispatcher` calls `readFrom()` after the radio delivers
raw bytes.

### Packet hashing

```cpp
void calculatePacketHash(uint8_t* dest_hash) const;
```

Computes an 8-byte SHA-256 truncation over `payload + payload_len + payload_type`.
This is what `MeshTables` stores for duplicate detection. The hash intentionally
excludes the path so that the same message arriving via two different routes
produces the same hash and is correctly identified as a duplicate.

### The `_snr` field

When `Dispatcher` receives a packet, it stores the raw SNR (×4, as a signed
byte) in `_snr`. `getSNR()` returns the floating-point value. This is used by
`Mesh` for the Rx-delay calculation and is included in TRACE packets for
path quality reporting.

---

## Identity

### `Identity` — verify-only

```cpp
class Identity {
public:
  uint8_t pub_key[PUB_KEY_SIZE];   // 32-byte Ed25519 public key
  // …
  bool verify(const uint8_t* sig, const uint8_t* message, int msg_len) const;
  bool matches(const Identity& other) const;
  int  copyHashTo(uint8_t* dest) const;   // copies PATH_HASH_SIZE (1) bytes
};
```

An `Identity` represents any party in the mesh — a remote node, a contact,
or a channel member — from whose public key you can:
1. **Verify signatures** (advertisement authenticity).
2. **Derive ECDH shared secrets** (when combined with a `LocalIdentity`).
3. **Compute the routing hash** — just the first byte(s) of `pub_key`.

### `LocalIdentity` — sign + ECDH

```cpp
class LocalIdentity : public Identity {
  uint8_t prv_key[PRV_KEY_SIZE];   // 64-byte Ed25519 private key (seed + pub)
  // …
  void sign(uint8_t* sig, const uint8_t* message, int msg_len) const;
  void calcSharedSecret(uint8_t* secret, const Identity& other) const;
};
```

`LocalIdentity` is the key pair held by *this* device. It can sign messages
(advertisements carry a signature over the advert payload) and perform ECDH.

**ECDH in MeshCore** uses Curve25519 (X25519). Because the key pairs are
Ed25519 (used for signing), the library internally converts the Ed25519 key
to an X25519 key before the Diffie-Hellman exchange. This is a well-established
technique (RFC 8037). The resulting 32-byte shared secret is then used as the
AES-128 key for payload encryption (via `Utils::encryptThenMAC` /
`Utils::MACThenDecrypt`).

### Key generation

```cpp
LocalIdentity(RNG* rng);          // generate a new random key pair
LocalIdentity(prv_hex, pub_hex);  // restore from hex strings
```

In firmware `setup()`, the identity is loaded from the `IdentityStore`
(filesystem-backed). If none exists, `radio_new_identity()` generates a fresh
one using hardware RNG seeded from the radio's noise floor.

The two reserved hash values `0x00` and `0xFF` are re-rolled if generated —
they are special-cased in routing logic.

### Persistence

`IdentityStore` (`src/helpers/IdentityStore.h`) handles reading and writing
`LocalIdentity` to the platform filesystem (LittleFS / SPIFFS / InternalFS).
The key is stored as raw bytes under a named key (e.g. `"_main"`).

---

## `Utils` — the crypto toolkit

`Utils` (`src/Utils.h`, `src/Utils.cpp`) provides stateless utility methods
used by `Identity`, `Mesh`, and application code:

| Method | Description |
|---|---|
| `sha256(hash, hash_len, msg, msg_len)` | Truncated SHA-256 |
| `encrypt(secret, dest, src, len)` | AES-128 CBC encryption (zero-padded final block) |
| `decrypt(secret, dest, src, len)` | AES-128 CBC decryption |
| `encryptThenMAC(secret, dest, src, len)` | AES-128 encrypt + prepend 2-byte MAC |
| `MACThenDecrypt(secret, dest, src, len)` | Verify 2-byte MAC, then decrypt; returns 0 on MAC failure |
| `toHex` / `fromHex` / `printHex` | Hex encode/decode/print utilities |
| `parseTextParts` | Split a string by separator (used by CLI parsers) |

The 2-byte MAC (`CIPHER_MAC_SIZE = 2`) in V1 packets is a truncated
SHA-256 over the ciphertext. It is deliberately short to minimise airtime;
the Ed25519 signatures on advertisements provide stronger authentication for
identity claims.

---

## Key takeaways

- **`Packet` is a plain struct with field accessors.** It is always in a pool;
  never `new`-allocated at runtime.
- **`Identity` is just a 32-byte public key + operations.** The routing hash
  is literally the first byte of that key — no separate hash field.
- **All crypto is in `Utils` and `Identity`.** The Mesh layer calls
  `encryptThenMAC` / `MACThenDecrypt` for every data payload; `Identity.sign`
  / `verify` for advertisements.
- **ECDH is Ed25519→X25519 converted.** One key pair does both signing and
  key agreement.

> **Cross-link:** Packet wire format (byte offsets, field sizes) →
> [`docs.meshcore.io/packet_format`](https://docs.meshcore.io/packet_format).
> Identity and encryption model (conceptual) →
> [Identity & Encryption](../concepts/identity-and-encryption.md).
