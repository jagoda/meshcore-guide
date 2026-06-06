# The Protocol

This section is a conceptual walkthrough of how MeshCore packets are
constructed, routed, and protected. The goal is to give you mental models
you can reason with — what each field *does*, why each design choice was
made, and what happens at every step of a message's life.

**Byte-level specs live elsewhere.** For exact field widths, bit positions, and
encoding tables, follow the cross-links to `docs.meshcore.nz` throughout each
page. This guide teaches the *why*; the specs supply the *what*.

---

## Reading order

| Page | What you'll learn |
|------|------------------|
| [Packet Journey](packet-journey.md) | End-to-end trace of a direct message from app to delivery |
| [Packet Anatomy](packet-anatomy.md) | The three sections of a MeshCore packet and what each field controls |
| [Payload Types Tour](payload-types-tour.md) | The full payload-type zoo — what each type is for and when it appears |
| [Routing and Flooding](routing-and-flooding.md) | How flood and direct routing interact; path discovery in depth |
| [Encryption on the Wire](encryption-on-the-wire.md) | Key agreement, MAC, and what is — and isn't — encrypted |
| [Adverts Deep Dive](adverts-deep-dive.md) | How node advertisements are built, signed, flooded, and consumed |

---

## Relation to docs.meshcore.nz

The upstream reference site documents the protocol at byte resolution:

- [Packet Format](https://docs.meshcore.nz/packet_format/) — header bit layout,
  `path_length` encoding, field sizes.
- [Payloads](https://docs.meshcore.nz/payloads/) — per-type payload structures.
- [Number Allocations](https://docs.meshcore.nz/number_allocations/) — registry
  for group data-type identifiers.

Each page in this section links out to the relevant spec at the point where
it crosses the conceptual/byte boundary.
