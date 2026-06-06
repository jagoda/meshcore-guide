# Core Concepts

The mental models that everything else in MeshCore builds on. If you are new to
MeshCore, read these pages before diving into the operator guides or the
protocol internals.

---

## Pages in this section

| Page | What you'll learn |
|------|-------------------|
| [Nodes and Roles](nodes-and-roles.md) | The five node roles (Companion, Repeater, Room Server, Sensor, Observer), why roles are fixed at flash time, and how to choose the right one. |
| [Adverts and Contacts](adverts-and-contacts.md) | How nodes announce their identity, how the contact list is built, and what path hashes are. |
| [How Messages Travel](how-messages-travel.md) | Flood routing, path discovery, direct routing, hop limits, and automatic fallback. |
| [Channels vs. Direct Messages](channels-vs-direct.md) | Point-to-point encrypted messages vs. group channels with a shared secret, and how Room Servers add persistence. |
| [Identity and Encryption](identity-and-encryption.md) | Ed25519 keys, ECDH shared secrets, AES-128 encryption, MACs, and why no central server is needed. |
| [Airtime and Regions](airtime-and-regions.md) | Duty cycle, LoRa parameters (BW/SF/CR), regional frequency bands, and how MeshCore's design keeps airtime low. |

---

## Recommended reading order

For **operators** (users running a personal node or small network): start with
Nodes and Roles → Adverts and Contacts → How Messages Travel → Channels vs.
Direct Messages, then continue to [Operating MeshCore →](../operating/index.md).

For **infrastructure operators** (deploying repeaters or room servers): read all
six pages — Airtime and Regions is especially important before you go on-air —
then continue to [Operating MeshCore →](../operating/index.md).

For **developers** (building on MeshCore or understanding the codebase): read
all six pages, then continue to [The Protocol →](../protocol/index.md) and
[Architecture & Internals](../internals/index.md).
