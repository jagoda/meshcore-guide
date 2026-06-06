# MeshCore Guide

Welcome to the MeshCore Guide — a narrative learning path for MeshCore, from
first boot to building your own integrations.

MeshCore is an open-source LoRa mesh networking platform for off-grid, encrypted
text communication — no internet, no cellular, no central server. This guide
teaches the **mental models and workflows** that make it click. Byte-level specs
— packet format, payload tables, companion protocol frames, CLI command catalogue
— live at [docs.meshcore.nz](https://docs.meshcore.nz); this guide cross-links
there rather than restating them.

## Who this guide is for

| You are… | Your starting point |
|----------|---------------------|
| **A new user** who just got a LoRa device | [Getting Started →](getting-started/index.md) |
| **An operator** setting up repeaters or room servers | Getting Started → Core Concepts → [Operating MeshCore](operating/index.md) |
| **A developer** building on or extending MeshCore | Getting Started → Core Concepts → [The Protocol](protocol/index.md) → [Companion API](companion-api/index.md) → [Internals](internals/index.md) |

## Two reading tracks

**Operator track** — start with [Getting Started](getting-started/index.md),
then [Core Concepts](concepts/index.md) and [Operating MeshCore](operating/index.md).

**Developer track** — start with [Getting Started](getting-started/index.md)
and [Core Concepts](concepts/index.md), then dive into
[The Protocol](protocol/index.md), [Companion API](companion-api/index.md),
and [Architecture & Internals](internals/index.md).

## This guide vs. the official spec docs

[docs.meshcore.nz](https://docs.meshcore.nz) is the authoritative **spec
reference** — exact packet formats, payload type tables, CLI command catalogue,
companion serial protocol frames. This guide is the **learning companion** that
explains *why* and *how* before pointing you at the exact bytes. Use both:
start here for concepts and workflows, follow the cross-links for the byte-level
detail.
