# Extending MeshCore

This section is the **destination of the arc**. Everything in the guide up to
here — flashing a device, understanding the mesh, operating infrastructure,
reading the protocol, using the Companion API, and studying the firmware
internals — has been preparation for this: building your own thing on top of
MeshCore.

This section is for developers who want to build on top of MeshCore — adding
new sensors, authoring custom payload applications, bringing up new hardware,
or integrating non-companion tooling via KISS. If you want a worked example
of what a complete host-side integration looks like before diving into firmware,
read the [Companion API — meshcore-ha walkthrough](../companion-api/worked-example-meshcore-ha.md)
first; it is the bridge between using the API and extending the firmware.

## What you can do

| Cookbook | When to use it |
|---|---|
| [Build from Source](build-from-source.md) | Compile your own firmware instead of using pre-built binaries. |
| [Custom Sensor](custom-sensor.md) | Wire a hardware sensor to stream readings over the mesh. |
| [Custom Application](custom-application.md) | Introduce a new payload data-type for a purpose-built mesh app. |
| [Custom Board Variant](custom-board-variant.md) | Add new hardware as a supported variant. |
| [KISS and Raw Packets](kiss-and-raw-packets.md) | Integrate MeshCore radios with TNCs, AX.25 stacks, or other radio software. |
| [Contributing Upstream](contributing-upstream.md) | Get your work merged into the official repository. |

## Prerequisites

All cookbooks assume:

- Comfort with C++ and the Arduino/PlatformIO ecosystem.
- Familiarity with the [Architecture & Internals](../internals/index.md) section — especially [Codebase Map](../internals/codebase-map.md) and [Example Apps](../internals/example-apps.md).
- The MeshCore source repo cloned locally.

## Relationship to Building Applications

There are two places to add application logic in MeshCore:

| Layer | Where code runs | Use when |
|---|---|---|
| **Host-side (Companion API)** | Desktop, SBC, phone | Logic runs on a capable machine; node is a radio modem |
| **Firmware-side (this section)** | On the MCU | Logic must run headless; you need access to routing tables, ACK callbacks, or sensor hardware |

If you haven't decided yet, read [Building Applications](../applications/index.md)
first — most use cases belong on the host side, and host-side iteration is
significantly faster.
