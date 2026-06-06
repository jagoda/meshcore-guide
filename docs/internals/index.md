# Architecture & Internals

This section is a developer-oriented tour of the MeshCore codebase.
If you want to understand how the firmware works under the hood — or you
want to extend it — start here.

## What you will find

| Page | What it covers |
|---|---|
| [Codebase Map](codebase-map.md) | Directory layout, key files, build system overview |
| [The Dispatcher](the-dispatcher.md) | The central scheduling loop, radio abstraction, packet queuing |
| [Mesh & Tables](mesh-and-tables.md) | Routing logic, flood/direct decisions, duplicate detection |
| [Packet & Identity](packet-and-identity.md) | The wire object, payload types, crypto primitives |
| [Boards & HAL](boards-and-hal.md) | `arch/`, `boards/`, `variants/`, the board abstraction layer |
| [Example Apps](example-apps.md) | The six `examples/` as readable templates and extension anchors |

## Prerequisites

- You are comfortable reading C++.
- You have PlatformIO installed (or at least understand the concept of a
  build environment per target board).
- You have read [Core Concepts](../concepts/index.md) — this section assumes
  you already know what a repeater, advert, and path hash are.

## The inheritance spine

Every firmware image is built on a single object hierarchy:

```
MainBoard          (hardware abstraction)
Radio              (radio abstraction)
PacketManager      (pool + outbound queue)
    │
Dispatcher         (Rx/Tx loop, duty-cycle budget)
    │
Mesh               (protocol routing, crypto dispatch)
    │
BaseChatMesh / SensorMesh / … (application logic per role)
    │
MyMesh             (example-specific overrides)
```

`Dispatcher` drives the radio in a cooperative `loop()` tick. `Mesh`
understands payload types and calls virtual `on…Recv()` callbacks.
Your `MyMesh` subclass fills those callbacks with application logic.
Nothing in this stack is multi-threaded; the whole thing runs in the
Arduino `loop()`.

---

**Where the arc continues:** Understanding the internals puts you in a position
to extend them. After this section, [Extending MeshCore →](../extending/index.md)
provides the practical cookbooks: adding a custom sensor, introducing a new
payload type, bringing up new hardware, and getting your work into the upstream
repository.
