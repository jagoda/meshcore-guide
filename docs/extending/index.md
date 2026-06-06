# Extending MeshCore

This section is for developers who want to build on top of MeshCore — adding new sensors, authoring custom payload applications, bringing up new hardware, or integrating non-companion tooling via KISS.

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
