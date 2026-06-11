# Glossary

Every term introduced in the MeshCore Guide narrative, alphabetical. Each entry gives a one-sentence definition and a pointer to the page where it is explained in context. Byte-level encodings are deferred to [`docs.meshcore.io`](https://docs.meshcore.io).

---

## A

**ACK (acknowledgement)**
A `PAYLOAD_TYPE_ACK` packet sent by the recipient's node after it successfully decrypts and delivers a direct message. As of v1.16 it is a 6-byte extended ACK: a 4-byte truncated SHA-256 of the original message (timestamp + text + sender public key), followed by the attempt number and a random byte that makes each ACK packet hash uniquely. The sender matches the 4-byte hash to confirm delivery.
→ [Packet Journey](../protocol/packet-journey.md), [Payload Types Tour](../protocol/payload-types-tour.md)

**Admin password**
A password that must be supplied before a CLI client can issue administrative commands (frequency changes, flood limits, etc.) to a Repeater or Room Server. The default is `password`; change it immediately on any deployed node.
→ [Running a Repeater](../operating/running-a-repeater.md)

**Advert / Advertisement**
A signed broadcast packet (`PAYLOAD_TYPE_ADVERT`) that contains a node's Ed25519 public key, name, type, timestamp, and optional GPS coordinates. Receiving nodes store adverts to build their contact list. Every advert is signed so it cannot be forged.
→ [Adverts and Contacts](../concepts/adverts-and-contacts.md), [Adverts Deep Dive](../protocol/adverts-deep-dive.md)

**AES-128**
The symmetric cipher used to encrypt direct-message and channel-message payloads on the wire. Key material is derived from an ECDH shared secret (direct messages) or a shared channel secret (group channels); never transmitted in the clear.
→ [Identity and Encryption](../concepts/identity-and-encryption.md), [Encryption on the Wire](../protocol/encryption-on-the-wire.md)

**Airtime**
The wall-clock duration a radio occupies the channel during a single LoRa transmission, determined by spreading factor, bandwidth, coding rate, and payload length. High-airtime transmissions consume a large share of the duty-cycle budget and increase collision risk for other nodes.
→ [Airtime and Regions](../concepts/airtime-and-regions.md)

**App data**
The variable-length extension field inside an advert payload that encodes node type (Companion/Repeater/Room Server/Sensor), name string, and GPS position. Receiving nodes parse this to populate the contact list.
→ [Adverts Deep Dive](../protocol/adverts-deep-dive.md)

**Application layer**
The layer above MeshCore's mesh transport where application logic lives — message types, reliability, session state, and addressing decisions. The Building Applications section is a guide to this layer.
→ Building Applications section

**ArduinoSerialInterface**
The `src/helpers/ArduinoSerialInterface` helper that implements the serial/BLE companion-protocol frame encoding and decoding on the node side; mirrors the framing described in the Frame Model page.
→ [Codebase Map](../internals/codebase-map.md), [Frame Model](../companion-api/frame-model.md)

---

## B

**BaseChatMesh**
The `src/helpers/BaseChatMesh` helper class that provides contact list management, message queuing, and ACK tracking used by the `companion_radio` example firmware.
→ [Codebase Map](../internals/codebase-map.md)

**BLE (Bluetooth Low Energy)**
One of the three transports over which the Companion API operates. In BLE mode the node exposes an NUS-compatible GATT service; the app writes commands to the RX characteristic and receives responses via TX notifications. One BLE characteristic write equals one companion protocol frame.
→ [Companion API — What It Is](../companion-api/what-it-is.md), [Frame Model](../companion-api/frame-model.md)

**BW (Bandwidth)**
The slice of radio spectrum a LoRa transmission occupies, in kHz. Common values: BW62.5, BW125, BW250. Narrower BW improves receiver sensitivity and reduces noise but increases airtime. All nodes on the same sub-mesh must use the same BW value.
→ [Airtime and Regions](../concepts/airtime-and-regions.md)

**build.sh**
A shell wrapper around PlatformIO CLI that lists available build targets, compiles one or all variants, and embeds the firmware version string. Primary entry point for building MeshCore firmware from source.
→ [Build from Source](../extending/build-from-source.md)

---

## C

**CayenneLPP**
A compact, self-describing binary format used by MeshCore sensor nodes to encode telemetry readings (temperature, humidity, GPS, voltage, etc.). Sensor nodes return CayenneLPP frames in response to `PAYLOAD_TYPE_REQ` stats requests.
→ [Sensors and Telemetry](../operating/sensors-and-telemetry.md), [Custom Sensor](../extending/custom-sensor.md)

**Channel secret**
A 16-byte AES key shared by all members of a group channel, distributed out-of-band (via QR code or secure copy). Any node that knows the channel secret can read and send group channel messages.
→ [Channels vs. Direct](../concepts/channels-vs-direct.md)

**ClientACL**
The `src/helpers/ClientACL` helper that maintains the authorised-client list for admin commands on Repeater and Room Server nodes; controls which keys may issue privileged CLI commands.
→ [Codebase Map](../internals/codebase-map.md), [Remote Admin CLI](../operating/remote-admin-cli.md)

**Coding Rate (CR)**
The proportion of error-correction redundancy added to a LoRa transmission (4/5, 4/6, 4/7, 4/8). Higher CR slightly increases airtime in exchange for better recovery from interference.
→ [Airtime and Regions](../concepts/airtime-and-regions.md)

**Collision storm**
A situation where poorly placed repeaters receive the same packet on multiple paths and all retransmit simultaneously, overwhelming the local channel. Mitigated by good repeater placement, tuning `flood.max`, and the built-in randomised retransmit delay.
→ [Airtime and Regions](../concepts/airtime-and-regions.md), [Routing and Flooding](../protocol/routing-and-flooding.md)

**CommonCLI**
The `src/helpers/CommonCLI` helper class that provides the admin CLI command parser used by Repeater, Room Server, and Sensor firmware examples.
→ [Codebase Map](../internals/codebase-map.md), [Remote Admin CLI](../operating/remote-admin-cli.md)

**Companion Radio**
The firmware role for a user-facing node: connects to a companion app over BLE, USB-Serial, or Wi-Fi; sends and receives direct and channel messages; does **not** retransmit packets for other nodes.
→ [Nodes and Roles](../concepts/nodes-and-roles.md), [What Is MeshCore?](../getting-started/what-is-meshcore.md)

**Companion API**
The serial/BLE frame API exposed by Companion Radio firmware; used by apps and tools (including meshcore-ha) to issue commands and receive responses and push notifications from a Companion Radio node.
→ [Companion API section](../companion-api/index.md)

**Companion protocol frame**
The fundamental unit of communication in the Companion API: a one-byte packet type followed by a variable-length payload, shared by commands (app→node) and responses (node→app).
→ [Frame Model](../companion-api/frame-model.md)

**Contact list**
The address book maintained by a Companion Radio node, populated from received adverts. Each entry holds the peer's public key, name, node type, last-seen time, and the cached ECDH shared secret for encryption.
→ [Adverts and Contacts](../concepts/adverts-and-contacts.md), [Send Your First Message](../getting-started/send-your-first-message.md)

**CR** → see *Coding Rate*

---

## D

**Default public channel**
A well-known channel key (`8b3387e9c5cdea6ac9e5edbaa115cd72`) baked into MeshCore as the global community channel. Treat it as a public square — assume no privacy. Any device configured with this key can participate in general community chat.
→ [Channels vs. Direct](../concepts/channels-vs-direct.md)

**Direct message**
A point-to-point encrypted message addressed to one specific contact, identified by public-key hash. Encrypted with the sender's ECDH shared secret so only the intended recipient can read it; uses flood routing on the first send, then direct routing after a path is returned.
→ [Channels vs. Direct](../concepts/channels-vs-direct.md), [Packet Journey](../protocol/packet-journey.md)

**Direct routing**
A routing mode in which the packet header carries an explicit ordered list of repeater hashes; only the listed repeaters forward the packet. Used for all subsequent messages after a path has been discovered; much more airtime-efficient than flooding.
→ [How Messages Travel](../concepts/how-messages-travel.md), [Routing and Flooding](../protocol/routing-and-flooding.md)

**Dispatcher**
The `src/Dispatcher` class that owns the radio, packet manager, and millisecond clock; implements the cooperative Rx/Tx scheduling loop. `Mesh` extends `Dispatcher` to add protocol awareness.
→ [The Dispatcher](../internals/the-dispatcher.md), [Codebase Map](../internals/codebase-map.md)

**Duty cycle**
The maximum fraction of time a radio may transmit, set by regional regulation. In EU868, typically 1% (36 s per rolling hour). MeshCore's design (companions don't repeat, direct routing reduces floods) naturally keeps duty-cycle usage low.
→ [Airtime and Regions](../concepts/airtime-and-regions.md)

---

## E

**ECDH (Elliptic-Curve Diffie–Hellman)**
The key-exchange protocol used to derive a shared secret from the sender's private key and the recipient's public key (using Ed25519 keys adapted to Curve25519/X25519 form). The shared secret is the basis for AES-128 encryption of direct messages.
→ [Identity and Encryption](../concepts/identity-and-encryption.md), [Encryption on the Wire](../protocol/encryption-on-the-wire.md)

**ECDH shared secret**
The 32-byte value derived by both parties to a direct-message exchange by running ECDH on their respective key pairs. Neither party transmits it; both compute the same value independently. Companion firmware caches it per-contact.
→ [Identity and Encryption](../concepts/identity-and-encryption.md)

**Ed25519**
The elliptic-curve signature scheme used for MeshCore node identity. Every node generates a 32-byte public key / 64-byte private key pair; adverts are signed with Ed25519 so recipients can verify the sender's identity without a certificate authority.
→ [Identity and Encryption](../concepts/identity-and-encryption.md), [Adverts and Contacts](../concepts/adverts-and-contacts.md)

**EU868**
The LoRa frequency plan for most of Europe (863–870 MHz, 1% duty cycle). Firmware must be flashed with a region-appropriate build; using EU868 hardware in a US915 region (or vice versa) degrades performance.
→ [Airtime and Regions](../concepts/airtime-and-regions.md), [Regions and Frequencies](regions-and-frequencies.md)

---

## F

**Flash advert** → see *Flood advert*

**flood.max**
A repeater CLI setting (`set flood.max <hops>`) that caps how many hops a channel flood is allowed to propagate through that node. Useful for containing community-channel traffic in dense meshes. As of v1.16, two companion settings cap specific flood classes separately: `flood.max.unscoped` (default 64, max 64) for unscoped/no-region floods and `flood.max.advert` (default 8, max 64) for advert floods.
→ [Channels vs. Direct](../concepts/channels-vs-direct.md), [Routing and Flooding](../protocol/routing-and-flooding.md)

**Flood advert**
An advert packet relayed by every repeater that receives it, propagating across the entire reachable mesh. Repeaters send one automatically every 47 hours by default (configurable). Companions can trigger a flood advert manually.
→ [Adverts and Contacts](../concepts/adverts-and-contacts.md), [Adverts Deep Dive](../protocol/adverts-deep-dive.md)

**Flood routing**
A routing mode in which every eligible repeater that has not yet seen the packet rebroadcasts it after a randomised delay. Used for the first message to a new contact, all group channel messages, and adverts. The seen-packet table prevents infinite loops.
→ [How Messages Travel](../concepts/how-messages-travel.md), [Routing and Flooding](../protocol/routing-and-flooding.md)

**Fragmentation**
Splitting a message that exceeds the 184-byte payload MTU into multiple sequenced packets, each carrying a portion of the payload. A `PAYLOAD_TYPE_MULTIPART` wrapper bundles multi-packet sequences at the protocol layer.
→ [Payload Types Tour](../protocol/payload-types-tour.md)

---

## G

**Group channel** → see *Channel secret*, *Channels vs. Direct*

**Guest password**
A password that Companion clients must supply to log in to a Room Server and retrieve stored posts. Default: `hello`; set with `set guest.password <pw>`.
→ [Running a Room Server](../operating/running-a-room-server.md), [Channels vs. Direct](../concepts/channels-vs-direct.md)

---

## H

**HAL (Hardware Abstraction Layer)**
The board-level glue code in `boards/`, `variants/`, and `arch/` that adapts the portable protocol stack to a specific MCU and radio chip. `MainBoard` is the interface; platform-specific implementations (`ESP32Board`, `NRF52Board`) live in `src/helpers/`.
→ [Boards and HAL](../internals/boards-and-hal.md)

**Header byte**
The first byte of every MeshCore packet, encoding three sub-fields: route type (flood vs. direct), payload type (what the content is), and payload version (wire-format convention). Repeaters read this byte to decide how to forward a packet without decrypting the payload.
→ [Packet Anatomy](../protocol/packet-anatomy.md)

**Hop**
One link traversal between two nodes on the mesh — from sender to the first repeater, or from one repeater to the next, or from the last repeater to the destination. A single message can traverse up to 64 hops.
→ [How Messages Travel](../concepts/how-messages-travel.md)

**Hop limit**
The maximum number of hops a flood packet may traverse before it is dropped (64). Encoded in the `path_length` byte; each repeater in the path decrements the hop count.
→ [How Messages Travel](../concepts/how-messages-travel.md), [Routing and Flooding](../protocol/routing-and-flooding.md)

---

## I

**Identity**
The `src/Identity` class that wraps a node's 32-byte Ed25519 public key and exposes a `verify()` method for checking signatures. Subclassed by `LocalIdentity`, which adds the private key and signing/ECDH capabilities.
→ [Packet and Identity](../internals/packet-and-identity.md), [Identity and Encryption](../concepts/identity-and-encryption.md)

**ISM band**
Industrial, Scientific, Medical — the unlicensed radio spectrum where LoRa operates. Key bands: 433 MHz (some Asian regions), 868 MHz (Europe), 915 MHz (Americas, Australia, NZ). No amateur radio licence is required for standard use, but duty-cycle rules still apply.
→ [Airtime and Regions](../concepts/airtime-and-regions.md)

---

## K

**Key hygiene**
The practice of keeping a node's private key off the air and backed up securely. If two devices share the same private key they are indistinguishable on the mesh. Private keys can be read/written over USB serial only.
→ [Identity and Encryption](../concepts/identity-and-encryption.md)

**KISS TNC**
A hardware/protocol interface (Keep It Simple Stupid Terminal Node Controller) that exposes raw LoRa packet framing to external software (Direwolf, APRSdroid, custom scripts). The `kiss_modem` example firmware implements this interface.
→ [KISS and Raw Packets](../extending/kiss-and-raw-packets.md)

---

## L

**LocalIdentity**
The `src/Identity::LocalIdentity` subclass that holds both the public and private key and exposes `sign()` and `calcSharedSecret()` methods. Each node has exactly one `LocalIdentity` at runtime.
→ [Packet and Identity](../internals/packet-and-identity.md), [Identity and Encryption](../concepts/identity-and-encryption.md)

**LoRa (Long Range)**
A radio modulation technique that trades data rate (low-kbps range) for range (multi-kilometre) and penetration. MeshCore uses LoRa in sub-GHz ISM bands. LoRa parameters (SF, BW, CR) determine airtime and range.
→ [What Is MeshCore?](../getting-started/what-is-meshcore.md), [Airtime and Regions](../concepts/airtime-and-regions.md)

---

## M

**MAC (Message Authentication Code)**
A 2-byte integrity check prepended to encrypted payloads. Computed by the sender over the ciphertext using the ECDH shared secret (direct messages) or channel secret (group). Receivers recompute the MAC; a mismatch means the packet is corrupt, forged, or addressed to the wrong node.
→ [Identity and Encryption](../concepts/identity-and-encryption.md), [Encryption on the Wire](../protocol/encryption-on-the-wire.md)

**MainBoard**
The abstract interface (`src/MeshCore.h`) that platform-specific board code must implement, providing radio hardware access, RTC, and system utilities. `ESP32Board` and `NRF52Board` are the two primary implementations.
→ [Boards and HAL](../internals/boards-and-hal.md)

**MAX_FRAME_SIZE**
176 bytes — the maximum size of a single companion protocol frame. Commands larger than this are rejected by the serial interface.
→ [Frame Model](../companion-api/frame-model.md)

**MAX_PATH_SIZE**
64 bytes — the maximum length of the path array in a packet header. Determines the maximum hop count as a function of hash size (64 hops at 1-byte hashes, 32 at 2-byte, 21 at 3-byte).
→ [Packet Anatomy](../protocol/packet-anatomy.md), [Routing and Flooding](../protocol/routing-and-flooding.md)

**Mesh**
The `src/Mesh` class that extends `Dispatcher` with MeshCore protocol knowledge: routing, payload-type dispatch, encryption/decryption, and the `on…Recv()` callback model. The central brain of the network stack.
→ [Mesh and Tables](../internals/mesh-and-tables.md), [Codebase Map](../internals/codebase-map.md)

**meshcore-ha**
The Home Assistant integration (`~/Documents/code/meshcore-ha`) — a real-world consumer of the Companion API, implemented with the meshcore-py SDK. Used in the Worked Example page as a concrete illustration of the connect→handshake→event-loop pattern.
→ [Worked Example — meshcore-ha](../companion-api/worked-example-meshcore-ha.md)

**MeshTables**
The abstract packet-table interface (implemented by `SimpleMeshTables`) that tracks recently seen packet hashes to suppress duplicates in flood routing and manage seen-path records.
→ [Mesh and Tables](../internals/mesh-and-tables.md)

**MTU (Maximum Transmission Unit)**
The maximum payload size for a single MeshCore packet: **184 bytes**. Larger messages must be fragmented via `PAYLOAD_TYPE_MULTIPART` or the application layer.
→ [Packet Anatomy](../protocol/packet-anatomy.md)

---

## N

**Node**
Any device running MeshCore firmware. Each node has exactly one role (Companion Radio, Repeater, Room Server, Sensor, or Observer mode), a unique Ed25519 key pair, and a configurable name.
→ [Nodes and Roles](../concepts/nodes-and-roles.md)

**Node type**
The role identifier embedded in an advert's app-data field: `1` = Companion (Chat), `2` = Repeater, `3` = Room Server, `4` = Sensor. Displayed in contact lists and used by apps to show appropriate UI.
→ [Adverts and Contacts](../concepts/adverts-and-contacts.md), [QR Codes and Links](qr-and-links.md)

**NUS (Nordic UART Service)**
The standard BLE GATT service profile used by MeshCore for BLE transport. Provides RX and TX characteristics for bidirectional frame exchange.
→ [Companion API — What It Is](../companion-api/what-it-is.md)

**Number allocations**
The registry of `PAYLOAD_TYPE_GRP_DATA` data-type values assigned to specific applications. Developer/testing range: `FF00–FFFF` (no registration needed). Production range: `0100–FEFF` (submit a PR to `docs/number_allocations.md` to reserve).
→ [Custom Application](../extending/custom-application.md), [Spec Index](spec-index.md)

---

## O

**Observer**
A passive monitoring mode available on Repeater firmware. An observer listens to all RF traffic and reports packet statistics to an aggregator (e.g., LetsMesh Analyser) but does not retransmit any packets.
→ [Nodes and Roles](../concepts/nodes-and-roles.md)

---

## P

**PacketManager**
The abstract packet-pool interface (`StaticPoolPacketManager` is the reference implementation) that manages a fixed pool of `Packet` objects and a priority outbound queue in the `Dispatcher`.
→ [The Dispatcher](../internals/the-dispatcher.md), [Codebase Map](../internals/codebase-map.md)

**Path**
The ordered list of repeater hashes embedded in a packet's header. In flood mode the path starts empty and each repeater appends its hash. In direct mode the full path is written at send time and each repeater removes its own hash as it forwards.
→ [How Messages Travel](../concepts/how-messages-travel.md), [Packet Anatomy](../protocol/packet-anatomy.md)

**Path discovery**
The process by which a Companion learns the route to a contact: the first message floods, and the recipient sends back a `PAYLOAD_TYPE_PATH` packet encoding the exact repeater sequence the message traversed. Subsequent messages use that path for direct routing.
→ [How Messages Travel](../concepts/how-messages-travel.md), [Routing and Flooding](../protocol/routing-and-flooding.md)

**Path expiry / fallback**
When a direct-routed message fails to deliver after retries, the sender clears the stored path and falls back to flooding. If flood delivery succeeds, the returned PATH updates the stored route automatically.
→ [How Messages Travel](../concepts/how-messages-travel.md)

**Path hash**
A 1-, 2-, or 3-byte prefix of a node's 32-byte Ed25519 public key, used as a compact identifier inside packet path fields. 1-byte hashes (the default) support 64 hops; 2- and 3-byte hashes require firmware ≥ 1.14 and improve disambiguation for analysis tools.
→ [Adverts and Contacts](../concepts/adverts-and-contacts.md), [Routing and Flooding](../protocol/routing-and-flooding.md)

**path_length byte**
A single byte in the packet header that encodes both the hash-size convention (1/2/3 bytes per hop entry) and the number of hops currently in the path. Repeaters decrement the hop count as they forward.
→ [Packet Anatomy](../protocol/packet-anatomy.md)

**PAYLOAD_TYPE_ACK**
Protocol constant (`0x03`): delivery acknowledgement. As of v1.16 the direct-message ACK is 6 bytes — a 4-byte truncated SHA-256 of the original message plus an attempt byte and a random byte. Unencrypted; any node can forward it.
→ [Payload Types Tour](../protocol/payload-types-tour.md)

**PAYLOAD_TYPE_ADVERT**
Protocol constant (`0x04`): node advertisement. Signed with Ed25519; not encrypted.
→ [Payload Types Tour](../protocol/payload-types-tour.md), [Adverts Deep Dive](../protocol/adverts-deep-dive.md)

**PAYLOAD_TYPE_ANON_REQ**
Protocol constant (`0x07`): an anonymous request that includes the sender's public key inline, allowing the recipient to derive a shared secret for the response without the sender being in the contact list.
→ [Payload Types Tour](../protocol/payload-types-tour.md)

**PAYLOAD_TYPE_CONTROL**
Protocol constant (`0x0B`): discovery and control packets, zero-hop only. Not relayed by repeaters.
→ [Payload Types Tour](../protocol/payload-types-tour.md)

**PAYLOAD_TYPE_GRP_DATA**
Protocol constant (`0x06`): group channel binary datagram, AES-encrypted with the channel secret. The 16-bit data-type sub-field identifies the application; custom apps use this type with a registered (or dev-range) data-type value.
→ [Payload Types Tour](../protocol/payload-types-tour.md), [Custom Application](../extending/custom-application.md)

**PAYLOAD_TYPE_GRP_TXT**
Protocol constant (`0x05`): group channel text message, AES-encrypted with the channel secret. Sender name is embedded but not signature-verified.
→ [Payload Types Tour](../protocol/payload-types-tour.md), [Channels vs. Direct](../concepts/channels-vs-direct.md)

**PAYLOAD_TYPE_MULTIPART**
Protocol constant (`0x0A`): wrapper for multi-packet sequences, used to bundle ACKs or to carry messages that exceed the 184-byte MTU.
→ [Payload Types Tour](../protocol/payload-types-tour.md)

**PAYLOAD_TYPE_PATH**
Protocol constant (`0x08`): a returned-path packet sent by the message recipient after a successful flood delivery. Contains the ordered list of repeater hashes the original flood traversed; the sender stores this path for subsequent direct sends.
→ [How Messages Travel](../concepts/how-messages-travel.md), [Payload Types Tour](../protocol/payload-types-tour.md)

**PAYLOAD_TYPE_RAW_CUSTOM**
Protocol constant (`0x0F`): opaque bytes for custom applications that do not fit the standard payload taxonomy.
→ [Payload Types Tour](../protocol/payload-types-tour.md)

**PAYLOAD_TYPE_REQ**
Protocol constant (`0x00`): a request from one node to a specific known peer (stats request, keepalive, custom). ECDH-encrypted.
→ [Payload Types Tour](../protocol/payload-types-tour.md)

**PAYLOAD_TYPE_RESPONSE**
Protocol constant (`0x01`): the reply to a `REQ` or `ANON_REQ`. ECDH-encrypted; content is application-defined.
→ [Payload Types Tour](../protocol/payload-types-tour.md)

**PAYLOAD_TYPE_TRACE**
Protocol constant (`0x09`): an SNR traceroute packet. Each repeater in the path records its receive SNR; the originating node collects the results to map signal quality hop-by-hop.
→ [Payload Types Tour](../protocol/payload-types-tour.md)

**PAYLOAD_TYPE_TXT_MSG**
Protocol constant (`0x02`): a direct text message. ECDH-encrypted; only the intended recipient can decrypt. Uses flood routing on first send, then direct routing after path discovery.
→ [Payload Types Tour](../protocol/payload-types-tour.md), [Packet Journey](../protocol/packet-journey.md)

**PAYLOAD_VER_1**
The current (and only deployed) payload version (`0x00` in the header's version bits): 1-byte source/destination hashes and a 2-byte MAC. Future versions may widen these fields.
→ [Packet Anatomy](../protocol/packet-anatomy.md)

**PlatformIO**
The build system and IDE plugin used to compile and flash MeshCore firmware. Targets are defined in `platformio.ini` and `variants/*/platformio.ini`; `build.sh` wraps the CLI.
→ [Build from Source](../extending/build-from-source.md), [Codebase Map](../internals/codebase-map.md)

**Private key**
The 64-byte Ed25519 secret that never leaves the device. Used to sign adverts and derive ECDH shared secrets. Can be backed up and restored via `get prv.key` / `set prv.key <hex>` over USB serial only.
→ [Identity and Encryption](../concepts/identity-and-encryption.md)

**Public key**
The 32-byte Ed25519 public key that serves as a node's permanent identity. Broadcast in every advert, stored in contact lists, and used to derive ECDH shared secrets for encryption. Never secret.
→ [Identity and Encryption](../concepts/identity-and-encryption.md), [Adverts and Contacts](../concepts/adverts-and-contacts.md)

---

## Q

**QR code**
A `meshcore://` URL encoded as a QR image. Used to share contact details (`meshcore://contact/add?…`) or channel keys (`meshcore://channel/add?…`) without requiring the recipient to be in RF range.
→ [QR Codes and Links](qr-and-links.md), [Adverts and Contacts](../concepts/adverts-and-contacts.md)

---

## R

**Randomised retransmit delay**
The random wait a repeater inserts before relaying a flood packet — a random multiple of half the estimated airtime for the packet, between 0× and 4×. Reduces simultaneous collision probability when multiple repeaters hear the same transmission.
→ [Routing and Flooding](../protocol/routing-and-flooding.md)

**RegionMap**
The `src/helpers/RegionMap` helper class that manages named RF regions and their transport keys, used for inter-region bridging via transport-code route types.
→ [Codebase Map](../internals/codebase-map.md), [Regions and Frequencies](regions-and-frequencies.md)

**Repeater**
The infrastructure firmware role. A Repeater forwards packets (both flood and direct) but has no user messaging interface. Typically deployed at elevation. Sends flood adverts every 47 hours by default.
→ [Nodes and Roles](../concepts/nodes-and-roles.md), [Running a Repeater](../operating/running-a-repeater.md)

**Returned path**
The list of repeater hashes carried in a `PAYLOAD_TYPE_PATH` packet sent by the message recipient after a flood delivery. The sending node stores this as the route for subsequent direct-routed messages to that contact.
→ [How Messages Travel](../concepts/how-messages-travel.md), [Routing and Flooding](../protocol/routing-and-flooding.md)

**Room Server**
A firmware role that acts as a persistent BBS: stores the last 32 posts and delivers them to clients when they log in with the guest password. For teams with intermittent connectivity.
→ [Nodes and Roles](../concepts/nodes-and-roles.md), [Running a Room Server](../operating/running-a-room-server.md), [Channels vs. Direct](../concepts/channels-vs-direct.md)

**ROUTE_TYPE_DIRECT**
A packet header route-type value indicating the packet carries an explicit repeater hash list and should be forwarded only by the listed repeaters.
→ [Packet Anatomy](../protocol/packet-anatomy.md)

**ROUTE_TYPE_FLOOD**
A packet header route-type value indicating the packet should be relayed by every eligible repeater that has not yet seen it.
→ [Packet Anatomy](../protocol/packet-anatomy.md)

**ROUTE_TYPE_TRANSPORT_DIRECT / ROUTE_TYPE_TRANSPORT_FLOOD**
Variants of the direct and flood route types that include a 4-byte transport-code block, used by transport repeaters that bridge independent RF regions.
→ [Packet Anatomy](../protocol/packet-anatomy.md)

---

## S

**Seen-packet table**
A cyclic ring-buffer (`SimpleMeshTables`, 160 entries) that tracks the hashes of recently received packets. If a repeater receives a packet whose hash is already in the table, it drops it silently — preventing flood loops.
→ [How Messages Travel](../concepts/how-messages-travel.md), [Routing and Flooding](../protocol/routing-and-flooding.md), [Mesh and Tables](../internals/mesh-and-tables.md)

**Sensor**
The firmware role for a telemetry node. Reports data (temperature, humidity, GPS, etc.) encoded as CayenneLPP frames in response to REQ packets. Does not participate in message routing.
→ [Nodes and Roles](../concepts/nodes-and-roles.md), [Sensors and Telemetry](../operating/sensors-and-telemetry.md), [Custom Sensor](../extending/custom-sensor.md)

**SensorManager**
The `src/helpers/SensorManager.h` abstract interface that sensor firmware subclasses to integrate hardware drivers. Provides `querySensors()`, `begin()`, `loop()`, and a settings registry for CLI read/write of sensor configuration.
→ [Custom Sensor](../extending/custom-sensor.md)

**SF (Spreading Factor)**
The LoRa parameter that controls how much each symbol is spread in time. Higher SF = longer range but dramatically more airtime. SF steps approximately double transmission time. Ranges from SF7 (fastest) to SF12 (slowest/longest range).
→ [Airtime and Regions](../concepts/airtime-and-regions.md)

**SimpleMeshTables**
The reference implementation of `MeshTables` — a fixed cyclic ring-buffer of 160 packet hashes used for duplicate detection in flood routing.
→ [Mesh and Tables](../internals/mesh-and-tables.md), [Codebase Map](../internals/codebase-map.md)

**SNR (Signal-to-Noise Ratio)**
A measure of received signal quality, logged by every repeater that forwards a packet. Exposed via the companion API's stats sub-commands and visible in traceroute (`PAYLOAD_TYPE_TRACE`) results. Helps operators diagnose weak links.
→ [Payload Types Tour](../protocol/payload-types-tour.md), [Common Operations](../companion-api/common-operations.md)

**Spreading Factor (SF)** → see *SF*

**StaticPoolPacketManager**
The reference `PacketManager` implementation: a fixed-size array of `Packet` objects and a priority-sorted outbound queue used by `Dispatcher`.
→ [The Dispatcher](../internals/the-dispatcher.md), [Codebase Map](../internals/codebase-map.md)

**Store-and-forward**
A messaging pattern where a relay node (typically a Room Server) accepts and holds messages for recipients who are not currently online, delivering them when the recipient next connects.
→ [Channels vs. Direct](../concepts/channels-vs-direct.md), [Running a Room Server](../operating/running-a-room-server.md)

---

## T

**TOFU (Trust On First Use)**
The trust model MeshCore uses for public keys: the first time a node's advert is received, the public key is accepted and stored. All subsequent packets from that node are verified against the stored key. If you need stronger assurance, verify the key in person or over a separate trusted channel.
→ [Identity and Encryption](../concepts/identity-and-encryption.md)

**Transport code**
A 4-byte block inserted after the header byte in `ROUTE_TYPE_TRANSPORT_*` packets, carrying a region-scoped identifier that bridge repeaters use to decide whether to relay a packet across a region boundary.
→ [Packet Anatomy](../protocol/packet-anatomy.md), [RegionMap](../internals/codebase-map.md)

---

## U

**US915**
The LoRa frequency plan for North America and much of the Americas and Oceania (902–928 MHz). No strict duty-cycle cap under FCC rules, but best practice is to minimise unnecessary transmissions.
→ [Airtime and Regions](../concepts/airtime-and-regions.md), [Regions and Frequencies](regions-and-frequencies.md)

**Utils**
The `src/Utils` module providing stateless crypto helpers: SHA-256, AES-128 encrypt/decrypt, ECDH, and hex encode/decode. Used throughout the stack; no Arduino dependency.
→ [Codebase Map](../internals/codebase-map.md)

---

## W

**Wire envelope**
The outer framing of an encrypted direct-message payload on the LoRa channel: `[dest_hash][src_hash][MAC (2 B)][AES-128 ciphertext]`. The hash bytes are used for routing lookups and are not encrypted; the ciphertext is opaque to all intermediate nodes.
→ [Packet Journey](../protocol/packet-journey.md), [Identity and Encryption](../concepts/identity-and-encryption.md)

---

## Z

**Zero-hop advert**
An advert broadcast once by the transmitter but not relayed by any repeater. Only nodes in direct RF range receive it. Used when you want to announce presence without consuming mesh airtime.
→ [Adverts and Contacts](../concepts/adverts-and-contacts.md), [Adverts Deep Dive](../protocol/adverts-deep-dive.md)
