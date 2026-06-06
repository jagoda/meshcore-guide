# Worked Example: Escape-Room / Scavenger-Hunt Infrastructure

**Patterns used:**
[message framing and type bytes](protocol-design-patterns.md#message-framing-and-type-bytes),
[request/response correlation](protocol-design-patterns.md#requestresponse-correlation),
[acks/retries/idempotency](protocol-design-patterns.md#acks-retries-and-idempotency),
[store-and-forward](protocol-design-patterns.md#store-and-forward),
[discovery and announce](protocol-design-patterns.md#discovery-and-announce),
[rate limiting and backoff](protocol-design-patterns.md#rate-limiting-and-backoff),
[protocol versioning](protocol-design-patterns.md#protocol-versioning)

**Code anchors:** [Custom Application](../extending/custom-application.md)
(firmware payload handler + actuator GPIO), [Custom Sensor](../extending/custom-sensor.md)
(reading a reed switch / RFID reader on the node), `examples/simple_room_server`
(store-and-forward relay model).

## What we're building

The **mesh as the wiring for a physical game** — an escape room or a
venue-scale scavenger hunt where the props *are* the network. Each puzzle
station is an embedded MeshCore node that:

- **Runs autonomously on the MCU.** A station fires its own electronic lock,
  LED strip, or solenoid the instant a team solves it — with **no host computer
  attached** to any prop. This is a firmware-side example by necessity: you
  cannot run a USB cable to a fake bookshelf.
- **Senses local input** — a keypad, RFID/NFC tag, reed switch, or button —
  and validates the answer on-device.
- **Reports progress** to a game-master node, buffering events when the GM is
  out of range so the leaderboard stays correct.

Teams carry a node (or a phone speaking the Companion API to a pocket node);
the team's public key *is* its identity. Stations are scattered across a
building where Wi-Fi and cell coverage are unreliable — exactly the niche LoRa
mesh fills well: low-bandwidth, low-power, infrastructure-free coverage of a
whole venue.

## Why this must be firmware-side

The defining requirement is **autonomous actuation**: a prop unlocks itself.
That demands logic on the node MCU:

- A **custom payload-type handler** that validates a claim and, on success,
  toggles a GPIO driving a relay/lock (see
  [Custom Application](../extending/custom-application.md)).
- **On-device input** read through the HAL (keypad, RFID — see
  [Custom Sensor](../extending/custom-sensor.md)).
- **Flash-persisted state** so a station remembers which teams have cleared it
  across a power blip mid-event.

Register a firmware payload type in the unassigned `0x0C–0x0E` range (verify
against `src/Packet.h` and the
[Number Allocations spec](https://docs.meshcore.io/number_allocations/)). Every
message carries an application-layer `type` byte inside the encrypted payload —
see [message framing and type bytes](protocol-design-patterns.md#message-framing-and-type-bytes).

## Architecture

```
            beacon (GRP_DATA flood)
Station ───────────────────────────►  every node in range (teams discover clues)
 node       claim (REQ unicast)
   ▲ ◄───────────────────────────────  Team node  (carried by players)
   │  claim-resp (RESPONSE: unlock!)
   │
   │ progress (REQ unicast, store-and-forward)
   ▼
Game-Master node  ──── state / leaderboard (GRP_DATA broadcast) ────►  all stations + teams
(operator host app, optional)
```

- **Stations** are firmware-side prop nodes. Each is a checkpoint.
- **Teams** carry a node; team logic can run host-side via the Companion API
  (a phone app) or on the carried node directly.
- The **game master** is one node, usually with an attached operator host app,
  that aggregates progress and broadcasts the leaderboard. The game runs
  *without* it — stations are autonomous; the GM is for scoring and oversight.

## Message types

Shared
[version + type framing](protocol-design-patterns.md#message-framing-and-type-bytes);
envelope inside the encrypted/PSK blob:

```
┌─────────┬──────┬──────────────────┬──────────────────────────────────┐
│ version │ type │ station_id (2 B) │ message payload (variable)       │
│  1 B    │  1 B │                  │                                  │
└─────────┴──────┴──────────────────┴──────────────────────────────────┘
```

`station_id` is a short venue-local identifier (configured at setup), not a
public key — there are only a handful of stations and the operator assigns them.
Team identity, by contrast, is the team node's 7-byte public-key prefix, carried
in claim payloads.

| Type byte | Name | Transport | Payload |
|---|---|---|---|
| `0x01` | `HUNT_BEACON` | GRP_DATA broadcast | station_id (2 B), clue text / hint (≤140 B) |
| `0x02` | `HUNT_CLAIM` | REQ unicast (team→station) | `claim_id` (4 B nonce), team_key (7 B), answer (≤80 B) |
| `0x03` | `HUNT_CLAIM_RESP` | RESPONSE (station→team) | `claim_id` echo (4 B), status (1 B), next clue (≤120 B) |
| `0x04` | `HUNT_PROGRESS` | REQ unicast (station→GM) | station_id (2 B), team_key (7 B), `event_seq` (2 B), timestamp (4 B) |
| `0x05` | `HUNT_PROGRESS_ACK` | RESPONSE (GM→station) | `event_seq` echo (2 B) |
| `0x06` | `HUNT_STATE` | GRP_DATA broadcast (GM) | leaderboard snapshot (≤150 B) |
| `0x07` | `HUNT_RESET` | REQ unicast (GM→station) | `reset_id` (4 B), new clue set version (1 B) |
| `0x00` | reserved | — | unknown/unrecognised (never valid) |

`status` values for `HUNT_CLAIM_RESP`: `0x00` accepted (and unlock fired),
`0x01` wrong answer, `0x02` already cleared by this team, `0x03` station locked
(prerequisite not met), `0x04` auth denied.

## Claiming a checkpoint: idempotency is the whole game

A team solves a station and sends `HUNT_CLAIM`. The station validates the
`answer`, and on success **fires the lock GPIO and records the clear**. The
`HUNT_CLAIM_RESP` carries the next clue. Now the failure mode that matters:
the response is lost, the team re-sends the claim, and a naive station
**fires the lock again and double-scores the team**.

This is the [acks/retries/idempotency](protocol-design-patterns.md#acks-retries-and-idempotency)
pattern, and here it is safety-relevant (a solenoid firing twice can jam a
mechanism):

- `claim_id` is a **random 32-bit nonce** from the team.
- The station stores, per team, the `claim_id` of the last accepted claim **and
  its `HUNT_CLAIM_RESP`**.
- On a duplicate `claim_id`, the station **re-sends the cached response and does
  not re-fire the actuator**. Status is the original `0x00`, so the team sees
  success exactly once in scoring terms.
- A team that has already cleared the station (different `claim_id`, same team,
  station already done) gets `status = 0x02 already cleared` — also without
  re-firing.

The actuator pulse is thus driven by the *state transition* (not-cleared →
cleared), never by packet arrival. The invariant: **a station fires its lock at
most once per team per game.**

## Authentication and anti-cheat

`HUNT_CLAIM` is a unicast `PAYLOAD_TYPE_REQ` encrypted with the ECDH shared
secret between team and station, so the station authenticates the **team_key**
cryptographically — a team cannot claim a checkpoint under another team's
identity without that team's private key. Additional measures:

- **Answer validation on-device.** The station holds the expected answer (or its
  hash); the answer never travels in the clear on a flood channel. Wrong answers
  return `status = 0x01` without leaking the correct one.
- **Replay protection.** The `claim_id` nonce plus the per-team last-claim record
  means a captured `HUNT_CLAIM` replayed by an eavesdropper is treated as a
  duplicate and re-returns the cached response — it cannot advance a team that
  has not actually solved the puzzle.
- **Prerequisite gating.** A station can require that the team has already
  cleared station *N* (checked against `HUNT_STATE` it has heard, or a local
  rule), returning `status = 0x03 locked` otherwise — enforcing puzzle order.

## Progress reporting with store-and-forward

The game master needs every clear for the leaderboard, but a station deep in a
basement may be out of range of the GM at the moment a team clears it. Dropping
that event corrupts scoring. The
[store-and-forward](protocol-design-patterns.md#store-and-forward) pattern
applies — here as **Option B, a custom firmware relay**, because the station
itself is the buffering node:

1. On a successful clear, the station enqueues a `HUNT_PROGRESS` event tagged
   with a monotonically increasing per-station `event_seq`.
2. The station transmits queued events to the GM and waits for
   `HUNT_PROGRESS_ACK` echoing the `event_seq`.
3. **Unacked events stay in the flash queue** and are retried with exponential
   [backoff](protocol-design-patterns.md#rate-limiting-and-backoff) (30 s → cap
   10 min) until acknowledged or the event ages out at end-of-game.
4. The GM **deduplicates on `(station_id, event_seq)`** so a retried event that
   was actually delivered the first time does not double-count.

Bound the queue depth per station (e.g. 32 events) and tag events with the game
session so a `HUNT_RESET` between groups discards stale ones. This is the same
TTL + dedup + bounded-queue discipline the
[store-and-forward](protocol-design-patterns.md#store-and-forward) section
prescribes.

## Discovery: beacons as clues

Stations announce themselves with `HUNT_BEACON` on a shared venue channel — the
[discovery and announce](protocol-design-patterns.md#discovery-and-announce)
pattern doing double duty as gameplay. A beacon's payload is the *public clue*
for that station ("find the cold one — temperature drops here"). Teams in range
receive it via channel flood and need no station public key to read it; the
station's identity for claiming is resolved from the `station_id` plus the
sender prefix of the beacon. Beacon cadence is slow and jittered (default once
per 2–3 minutes) to respect the [duty cycle](protocol-design-patterns.md#rate-limiting-and-backoff)
on a channel shared by many stations.

## Duty-cycle accounting

A venue with 10 stations and 6 teams, mid-event:

| Event | Packets | Frequency |
|---|---|---|
| `HUNT_BEACON` | 10 (one per station) | 1 per station per ~2.5 min |
| `HUNT_CLAIM` + resp | 2 | bursty, ~1 solve per team per few min |
| `HUNT_PROGRESS` + ack | 2 | 1 per solve |
| `HUNT_STATE` | 1 (flood) | 1 per minute |

Beacons dominate steady-state at ~4 packets/minute across all stations; total
load stays well under 10 packets/minute. **Stagger station beacon phases** at
setup so they do not all transmit in the same window — otherwise 10 simultaneous
floods collide. Audit against your regional limit per
[Airtime and Regions](../concepts/airtime-and-regions.md); for a one-evening
event the average is comfortable, but the synchronized-beacon failure mode is
the thing to design out.

## What the mesh makes hard (and how we lean in)

- **Latency is seconds.** A lock that fires 3–8 seconds after a correct answer
  is fine for an escape room (it reads as dramatic tension, not lag). Tell the
  player "transmitting…" so the delay is legible.
- **Coverage is uneven.** That is exactly why a *mesh* and not point-to-point:
  stations relay for each other, and store-and-forward covers the GM blind
  spots. Place at least one repeater so no station is fully isolated.
- **Packets are lost.** Idempotent claims and acked-with-dedup progress mean a
  lossy room still scores correctly — liveness degrades gracefully, correctness
  does not.

## Cross-links

- [Custom Application](../extending/custom-application.md) — firmware payload
  handler and driving an actuator GPIO on a clear.
- [Custom Sensor](../extending/custom-sensor.md) — reading keypads / RFID on the
  station node.
- [Running a Repeater](../operating/running-a-repeater.md) — covering venue
  blind spots.
- [Payload Types Tour](../protocol/payload-types-tour.md) — REQ/RESPONSE and
  GRP_DATA in context.
- [Protocol Design Patterns](protocol-design-patterns.md) — all patterns
  referenced above.
- [Building and Testing](building-and-testing.md) — bench-testing a station's
  claim/unlock path without RF.
