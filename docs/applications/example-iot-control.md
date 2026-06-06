# Worked Example: IoT Remote Control

**Patterns used:**
[message framing and type bytes](protocol-design-patterns.md#message-framing-and-type-bytes),
[request/response correlation](protocol-design-patterns.md#requestresponse-correlation),
[acks/retries/idempotency](protocol-design-patterns.md#acks-retries-and-idempotency),
[rate limiting and backoff](protocol-design-patterns.md#rate-limiting-and-backoff),
[discovery and announce](protocol-design-patterns.md#discovery-and-announce)

**Code anchors:** `examples/companion_radio` (host-side model),
`examples/simple_secure_chat` (REQ/RESPONSE unicast pattern).

## What we're building

A remote-control application for battery-powered IoT devices on the mesh:

- A **controller host app** (running on a Pi or laptop) sends commands to
  named devices.
- Each device runs a **Companion Radio** node that relays commands to a small
  microcontroller (the actuator) via a local serial or I²C link, or runs
  the actuator logic directly on the radio MCU via a custom firmware task.
- Devices push **telemetry** (sensor readings) at a configurable interval.
- Commands are **idempotent**: a duplicate command is safe to execute twice.
- The system is designed to stay within legal duty-cycle limits even when
  multiple devices and a controller are on the same mesh.

## Architecture

```
Controller host app
    │ Companion API (USB serial)
    ▼
Companion Radio node ──[LoRa]──► Device Companion Radio node
                                     │ local serial
                                     ▼
                                   Actuator MCU
```

All mesh logic is unicast (`PAYLOAD_TYPE_REQ` / `PAYLOAD_TYPE_RESPONSE`),
addressed to the device's public key. Telemetry is multicast via a private
channel using `PAYLOAD_TYPE_GRP_DATA`.

## Device addressing and discovery

Devices advertise with `ADV_TYPE_CHAT` (or a custom type if you build custom
firmware) and embed an application type tag in their advert name field:
e.g., `PUMP-01` or a structured binary blob in the advert data field
(up to `MAX_ADVERT_DATA_SIZE` = 32 bytes — see
[Adverts Deep Dive](../protocol/adverts-deep-dive.md)).

The controller scans its contact list for devices matching the expected name
prefix or advert data pattern. Once a device's public key is known, all
subsequent communication is direct unicast.

## Command protocol

Commands travel as `PAYLOAD_TYPE_REQ` packets (unicast, encrypted,
end-to-end authenticated). The application payload inside the REQ:

```
┌─────────┬──────┬────────────────┬──────────────────────────────────┐
│ version │ type │  cmd_id (4 B)  │ command payload (variable)       │
│  1 B    │  1 B │                │                                  │
└─────────┴──────┴────────────────┴──────────────────────────────────┘
```

### Command types

| Type byte | Name | Payload |
|---|---|---|
| `0x01` | `CMD_SET` | key (1 B), value (up to ~155 B) |
| `0x02` | `CMD_GET` | key (1 B) |
| `0x03` | `CMD_ACTION` | action_id (1 B), params (up to ~155 B) |
| `0x80` | `RESP_OK` | `cmd_id` echo (4 B), optional result |
| `0x81` | `RESP_ERR` | `cmd_id` echo (4 B), error code (1 B), message |

### Idempotency

The `cmd_id` is a **random 32-bit nonce** chosen by the controller at command
construction time. The device stores the last N (e.g., 8) `cmd_id` values it
has executed in a circular buffer. On receipt:

1. If `cmd_id` is in the buffer, re-send the cached `RESP_OK` without
   re-executing the action.
2. If `cmd_id` is new, execute, add to buffer, send `RESP_OK`.

This makes it safe for the controller to retry a timed-out command without
the risk of double-execution (e.g., toggling a relay twice).

### Retry policy

```
attempt 1 → wait 30 s
attempt 2 → wait 60 s
attempt 3 → wait 120 s
attempt 4 → give up, surface error to operator
```

Cap at 3 retries to avoid flooding the channel when a device is genuinely
offline. Report the failure; let the operator decide whether to retry manually.

## Authorization

Because unicast REQ packets are encrypted with the ECDH shared secret between
controller and device, only the controller (whose public key the device trusts)
can send authenticated commands. However, **any node on the mesh can send a
REQ to the device** if it discovers the device's public key.

Mitigations:

1. **Allowlist by public key.** The device firmware checks the sender's public
   key prefix against a configurable allowlist. Unknown senders receive
   `RESP_ERR` with error code `AUTH_DENIED`.
2. **Application-layer token.** Include a 16-byte pre-shared token in
   `CMD_ACTION` payloads for sensitive operations. The device verifies it
   before acting.

For most hobby deployments, the ECDH encryption is sufficient — an attacker
would need to break Curve25519 to forge a command. The allowlist is a
defence-in-depth measure for safety-critical actuators.

## Push telemetry

Devices publish sensor readings at a configurable interval as `PAYLOAD_TYPE_GRP_DATA`
on a private monitoring channel (a pre-shared key distributed out-of-band to
all controllers):

```
┌─────────┬──────┬──────────────────────┬─────────────────────┐
│ version │ type │ device_key (7 B)     │ CayenneLPP payload  │
│  1 B    │  1 B │ (sender pub prefix)  │ (up to ~150 B)      │
└─────────┴──────┴──────────────────────┴─────────────────────┘
```

[CayenneLPP](https://developers.mydevices.com/cayenne/docs/lora/#lora-cayenne-low-power-payload)
is a compact binary encoding for sensor data used throughout the MeshCore
ecosystem (see `examples/simple_room_server/MyMesh.h` for its `CayenneLPP`
usage). It encodes typed channels (temperature, humidity, voltage, etc.) in a
self-describing format that is easy to parse in any language.

Type byte for telemetry: `0x10` (push, unsolicited) and `0x11` (response to
`CMD_GET`).

## Duty-cycle-aware control loops

**Do not poll.** Polling (send a GET, wait for response, repeat) is the most
common source of duty-cycle violations in IoT control applications. Instead:

1. **Push by default.** Devices send telemetry on their own schedule; the
   controller listens.
2. **Adjust the push interval dynamically.** The controller can send a
   `CMD_SET key=TELEMETRY_INTERVAL value=<seconds>` to change the push rate.
3. **Poll only on demand.** If the operator explicitly requests a fresh reading,
   send `CMD_GET`. Do not put `CMD_GET` in a loop.
4. **Back off on silence.** If a device has not sent telemetry in 3× its
   expected interval, issue one `CMD_GET`. If still silent, mark the device as
   offline and alert the operator — do not keep polling.

Telemetry packets are small (~20–60 bytes of CayenneLPP typically). At one
packet per 5 minutes, 10 devices consume roughly 2 packets/minute = well under
any duty-cycle budget. At one packet per 30 seconds per device, 10 devices is
20 packets/minute — audit your airtime budget carefully before going there.

## Implementation surface decision

This example is designed as a **hybrid**:

- Command routing and logic live on a **host-side controller app** using the
  Companion API.
- The device side can be either:
  - A Companion Radio node relaying commands to a separate actuator MCU
    (no firmware changes), or
  - Custom firmware on the radio MCU that implements the command handler
    directly (see [Custom Application](../extending/custom-application.md)).

The hybrid approach is recommended for early-stage projects: prototype the
protocol host-side first, then move the device handler into firmware only if
the system genuinely needs to run without a host attached.

## Cross-links

- [Companion API § Common Operations](../companion-api/common-operations.md) —
  sending REQ frames from the host app.
- [Payload Types Tour](../protocol/payload-types-tour.md) — REQ/RESPONSE and
  GRP_DATA in context.
- [Adverts Deep Dive](../protocol/adverts-deep-dive.md) — device discovery.
- [Airtime and Regions](../concepts/airtime-and-regions.md) — duty-cycle
  budget for your region.
- [Protocol Design Patterns](protocol-design-patterns.md) — all patterns
  referenced above.
- [Building and Testing](building-and-testing.md) — testing without RF.
