# Building Applications

MeshCore ships with firmware for chat nodes, room servers, repeaters, and
home-automation gateways. But the protocol is open: nothing stops you from
building your own application that uses the mesh as a transport layer. This
section is a developer's guide to doing that well.

## What "an application protocol on MeshCore" means

When you build on MeshCore you are working at the **application layer** — above
the mesh transport but below user experience. Your application defines:

- **Message types** — what bytes mean what.
- **Addressing** — which node(s) should receive a given message.
- **Reliability** — what happens when a packet is lost or delayed.
- **Session state** — how multi-step interactions are tracked across the
  async, lossy medium.

MeshCore's job is to move encrypted packets between nodes. Your application's
job is to put useful semantics on top of those packets.

## The layering model

```
┌──────────────────────────────────────────┐
│           Your Application               │  ← this section
│  (message types, state, reliability)     │
├──────────────────────────────────────────┤
│       MeshCore Payloads / Frames         │  ← Protocol §4, Companion API §5
│  (TXT_MSG, REQ/RESPONSE, GRP_DATA, …)   │
├──────────────────────────────────────────┤
│         Mesh Transport                   │  ← Protocol §4
│  (flood/direct routing, ACKs, path)      │
├──────────────────────────────────────────┤
│            LoRa PHY                      │
│  (airtime, RSSI, duty cycle)             │
└──────────────────────────────────────────┘
```

Each layer has a dedicated spec on [docs.meshcore.nz](https://docs.meshcore.nz).
This guide does not re-derive those specs; it teaches how to *use* them
together to ship a real application.

## Two implementation surfaces

You have two places to add application code:

| Surface | Where your code runs | Mechanism | When to choose |
|---|---|---|---|
| **Host-side companion app** | Desktop / SBC / phone | Companion API frames over serial/BLE | Most applications; no firmware build needed; easiest iteration |
| **Firmware-side custom payload type** | On the node (MCU) | Register a new `PAYLOAD_TYPE_*` handler in `Mesh.h` | Need to run on the node itself (relay logic, sensor aggregation without a host) |

The vast majority of applications belong on the **host side**. If your
application needs a Companion Radio node as the radio modem and runs the logic
on a more capable machine (Pi, laptop, phone), use the Companion API. If you
genuinely need the logic to live *inside* the firmware, see
[Custom Application](../extending/custom-application.md) in Extending MeshCore.

The [design-constraints page](design-constraints.md) helps you understand the
envelope before you commit to either surface.

## The three worked examples

This section is anchored by three applications at different points of the
design space:

| Example | Class | Key patterns used |
|---|---|---|
| [Email-like messaging](example-messaging.md) | Store-and-forward mailbox | offline delivery, delivery receipts, room-server relay |
| [IoT remote control](example-iot-control.md) | Command/response | idempotent commands, push telemetry, duty-cycle-aware polling |
| [Multiplayer game](example-multiplayer-game.md) | Turn-based state sync | turn sequencing, lobby discovery, trust/anti-cheat |

Each example maps its design choices back to the shared vocabulary in
[Protocol Design Patterns](protocol-design-patterns.md), so you can mix
and match patterns for your own application class.

## Before you build: the hard constraints

LoRa is a slow, lossy, half-duplex radio channel shared with every other node
in range. Constraints that are easy to paper over in TCP/IP applications
are fundamental here:

- **MTU is 184 bytes** (raw payload field; your usable application payload is
  smaller once framing overhead is removed — see
  [Design Constraints](design-constraints.md)).
- **No guaranteed delivery.** You own reliability.
- **Airtime is a shared resource.** Duty-cycle limits are legal in most
  regions; violating them makes you a bad neighbour.
- **Latency is measured in seconds, not milliseconds.** Multi-hop routes add
  per-hop retransmit delays.
- **Connectivity is intermittent.** Nodes sleep, move, go out of range.

These are not obstacles; they are design inputs. The patterns and examples
that follow show how to work *with* these constraints rather than against them.

## Where to go next

New to MeshCore application development? Read the pages in order:

1. [Design Constraints](design-constraints.md) — understand the envelope.
2. [Protocol Design Patterns](protocol-design-patterns.md) — your toolkit.
3. Pick the worked example closest to your use case.
4. [Building and Testing](building-and-testing.md) — dev loop and testing
   without live RF.
