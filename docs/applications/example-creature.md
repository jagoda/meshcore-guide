# Worked Example: A Creature That Lives in the Mesh

**Patterns used:**
[message framing and type bytes](protocol-design-patterns.md#message-framing-and-type-bytes),
[request/response correlation](protocol-design-patterns.md#requestresponse-correlation),
[acks/retries/idempotency](protocol-design-patterns.md#acks-retries-and-idempotency),
[discovery and announce](protocol-design-patterns.md#discovery-and-announce),
[sequencing and ordering](protocol-design-patterns.md#sequencing-and-ordering),
[rate limiting and backoff](protocol-design-patterns.md#rate-limiting-and-backoff),
[protocol versioning](protocol-design-patterns.md#protocol-versioning)

**Code anchors:** [Custom Application](../extending/custom-application.md)
(firmware-side payload handler), `examples/simple_secure_chat`
(REQ/RESPONSE unicast pattern), `examples/companion_radio`
(`DataStore` persistence model).

## What we're building

A **digital creature whose home is the mesh itself** — a small autonomous agent
that lives *inside node firmware*, not on an attached host. The creature:

- **Runs on the node MCU.** It ticks, ages, and reacts to mesh events even when
  no phone or laptop is connected. This is a deliberately **firmware-side**
  example (contrast the host-side messaging/IoT applications).
- **Has internal state** — energy, mood, and a memory of which nodes it has
  visited — persisted to flash so it survives reboots and battery swaps.
- **Migrates between nodes.** Exactly one node *hosts* the creature at any
  moment; it hands itself off to a neighbour, so it appears to roam the physical
  mesh like a Tamagotchi with wanderlust.
- **Lets people interact with it** — feed it, play with it, check on it — from
  any node in range, host-side or firmware-side.

This is the canonical "do I really need firmware?" example: the creature cannot
live on a host, because the whole point is that it persists and acts on the node
with nothing attached.

## Why this must be firmware-side

A host-side application stops the moment the host disconnects. The creature's
identity is that it is *always alive somewhere in the mesh*. That requires:

- An **on-MCU tick loop** that decays energy/mood on a timer, independent of any
  host.
- **Flash persistence** so state survives power loss (use the same
  `DataStore`-style pattern `companion_radio` uses for contacts).
- A **custom payload-type handler** registered in firmware so the node can act
  on creature packets autonomously.

Register a firmware payload type in the unassigned `0x0C–0x0E` range (check
`src/Packet.h` and the
[Number Allocations spec](https://docs.meshcore.io/number_allocations/) before
choosing) — see
[message framing and type bytes](protocol-design-patterns.md#message-framing-and-type-bytes).
All examples below carry an application-layer `type` byte inside the encrypted
payload; the mesh layer never inspects it.

## Architecture

```
        feed / play (unicast REQ)                heartbeat (GRP_DATA flood)
People ───────────────────────────►  Host Node  ─────────────────────────►  every node
                                     (creature lives here, ticks on MCU)      on the creature channel
                                          │
                                          │ migration handoff (two-phase REQ/RESPONSE)
                                          ▼
                                     Neighbour Node
                                     (becomes new host)
```

- **Interactions** are unicast `PAYLOAD_TYPE_REQ` / `PAYLOAD_TYPE_RESPONSE` to
  whichever node currently hosts the creature — encrypted and authenticated by
  ECDH, so the creature knows *who* fed it.
- **Heartbeats** are `PAYLOAD_TYPE_GRP_DATA` broadcast on a shared "creature"
  channel so spectators can watch its mood without holding its public key.
- **Migration** is a two-phase unicast handoff between the current host and a
  chosen neighbour — the critical correctness mechanism (see below).

## The single-authority invariant

The hardest part of "a creature that lives in the mesh" is the same problem the
[multiplayer game](example-multiplayer-game.md) solves for its host role:
**there must be exactly one authoritative copy of the creature at any time.**
Two hosts means a forked creature — duplicated, divergent, and able to be fed
twice. The invariant: *the creature is a token; only the token-holder may mutate
or migrate it.*

Migration is therefore a **two-phase handoff**, not a fire-and-forget transfer:

1. Current host **A** picks a neighbour **B** (highest-RSSI recent contact) and
   sends `CREA_MIGRATE_OFFER` (REQ) carrying a `migration_id` nonce.
2. **B** replies `CREA_MIGRATE_ACCEPT` (RESPONSE) echoing the `migration_id`.
3. **A** sends `CREA_MIGRATE_COMMIT` (REQ) containing the full authoritative
   state snapshot, then **immediately marks itself dormant** — it will no longer
   tick or accept interactions for that creature.
4. **B** persists the snapshot, replies `CREA_MIGRATE_DONE`, and begins ticking.

If step 3 or 4 is lost, **A** is dormant and **B** never started: the creature
appears frozen, but is *not duplicated*. Recovery: if **A** gets no
`CREA_MIGRATE_DONE` within the retry window, it **resumes from its
pre-migration snapshot** (it kept it) and retries with a fresh `migration_id`.
This is the [acks/retries/idempotency](protocol-design-patterns.md#acks-retries-and-idempotency)
pattern: prefer a temporarily-frozen creature over a forked one. Losing liveness
is recoverable; losing the single-authority invariant is not.

## Message types

All messages share the standard
[version + type framing](protocol-design-patterns.md#message-framing-and-type-bytes).
Envelope inside the encrypted blob:

```
┌─────────┬──────┬────────────────────┬──────────────────────────────────┐
│ version │ type │ creature_id (7 B)  │ message payload (variable)       │
│  1 B    │  1 B │ (creature pub key) │                                  │
└─────────┴──────┴────────────────────┴──────────────────────────────────┘
```

The creature owns its own Curve25519 keypair, generated once and migrated with
its state. `creature_id` is the 7-byte public-key prefix — a stable identity
that does not change as the creature roams between physical nodes.

| Type byte | Name | Transport | Payload |
|---|---|---|---|
| `0x01` | `CREA_HEARTBEAT` | GRP_DATA broadcast | `state_seq` (2 B), mood (1 B), energy (1 B), host_key prefix (7 B) |
| `0x02` | `CREA_INTERACT` | REQ unicast | `act_id` (4 B nonce), action (1 B), params (variable) |
| `0x03` | `CREA_INTERACT_RESP` | RESPONSE | `act_id` echo (4 B), status (1 B), new mood (1 B), new energy (1 B) |
| `0x04` | `CREA_MIGRATE_OFFER` | REQ unicast | `migration_id` (4 B nonce), `state_seq` (2 B) |
| `0x05` | `CREA_MIGRATE_ACCEPT` | RESPONSE | `migration_id` echo (4 B) |
| `0x06` | `CREA_MIGRATE_COMMIT` | REQ unicast | `migration_id` (4 B), full state snapshot |
| `0x07` | `CREA_MIGRATE_DONE` | RESPONSE | `migration_id` echo (4 B) |
| `0x00` | reserved | — | unknown/unrecognised (never valid) |

`action` values for `CREA_INTERACT`: `0x01` feed, `0x02` play, `0x03` pet,
`0x04` observe (read-only). Unknown actions return `status = ERR_UNKNOWN_ACTION`.

## State model and the heartbeat

The authoritative state is a compact struct that must fit in a single migration
packet:

```
┌───────────┬──────┬────────┬───────────┬──────────────────────────────┐
│ state_seq │ mood │ energy │ born_at   │ visited (ring of node prefixes)│
│   2 B     │ 1 B  │  1 B   │   4 B     │  up to 16 × 7 B               │
└───────────┴──────┴────────┴───────────┴──────────────────────────────┘
```

`state_seq` is a monotonically increasing 16-bit counter bumped on **every**
state mutation (a feed, a tick decay, a migration). It is the creature's
logical clock, used exactly as the
[sequencing and ordering](protocol-design-patterns.md#sequencing-and-ordering)
pattern prescribes:

- Spectators track the highest `state_seq` they have seen in a `CREA_HEARTBEAT`.
  A heartbeat with a *lower* `state_seq` is a stale straggler (arrived late via a
  longer flood path) and is **dropped**.
- The receiving host of a migration rejects a `CREA_MIGRATE_COMMIT` whose
  `state_seq` is not newer than the last it holds — replay protection for the
  handoff.

The host emits `CREA_HEARTBEAT` on a **slow** cadence (default once per
5 minutes), jittered. This is the
[rate limiting and backoff](protocol-design-patterns.md#rate-limiting-and-backoff)
discipline: a creature that chatters is a bad mesh neighbour. Mood/energy decay
happens on the MCU tick regardless of heartbeat rate; the heartbeat only
*announces* state, it does not drive it.

## Interaction and idempotency

Feeding is the textbook case for
[idempotency](protocol-design-patterns.md#acks-retries-and-idempotency): a lost
`CREA_INTERACT_RESP` must not let an anxious user feed the creature five times.

- `act_id` is a **random 32-bit nonce** chosen by the interacting node.
- The host keeps a small circular buffer of the last N (e.g. 8) `act_id`s it has
  applied, per sender.
- On a duplicate `act_id`, the host **re-sends the cached `CREA_INTERACT_RESP`
  without re-applying the action**. The creature's energy is unchanged.

Because interactions are unicast REQ packets encrypted with the ECDH shared
secret, the creature authenticates *who* is interacting — enabling per-visitor
memory ("this node has fed me before") without any extra auth layer, the same
property the [IoT example's authorization](protocol-design-patterns.md#requestresponse-correlation)
section relies on.

## Discovery

A node that wants to find the creature uses
[discovery and announce](protocol-design-patterns.md#discovery-and-announce):
it subscribes to the creature channel and waits for a `CREA_HEARTBEAT`, which
carries the current `host_key` prefix. All subsequent interactions are unicast
to that host. When the creature migrates, the next heartbeat advertises the new
host automatically — interactors re-target on the updated `host_key` with no
explicit "the creature moved" message required.

## Duty-cycle accounting

A single creature on a 20-node mesh:

| Event | Packets | Frequency |
|---|---|---|
| `CREA_HEARTBEAT` | 1 (flood) | 1 per 5 min |
| `CREA_INTERACT` + resp | 2 | bursty, human-driven (rare) |
| Migration (4 packets) | 4 | at most a few per hour |

Steady-state is well under 1 packet/minute — trivially within any regional duty
cycle. The migration burst is the only spike; cap migrations to at most once per
N minutes (the creature does not need to teleport constantly) to keep the
average flat. See [Airtime and Regions](../concepts/airtime-and-regions.md).

## What the mesh makes hard (and how we lean in)

- **No global clock.** The creature ages on each host's local MCU tick.
  Migration carries `born_at` so age is continuous; per-host clock skew of a few
  seconds is cosmetically invisible for a pet.
- **Packets are lost.** The two-phase migration is built around this: the
  single-authority invariant holds *because* we assume the network drops the
  commit.
- **Nodes sleep and move.** That is the feature, not the bug — it is what makes
  the creature feel like it lives somewhere and wanders. The slow heartbeat and
  flash persistence mean a sleeping host simply resumes the creature on wake.

## Cross-links

- [Custom Application](../extending/custom-application.md) — registering the
  firmware payload-type handler and the on-MCU tick loop.
- [Multiplayer Game](example-multiplayer-game.md) — the same single-authority /
  host-token problem in a different shape.
- [Payload Types Tour](../protocol/payload-types-tour.md) — REQ/RESPONSE and
  GRP_DATA in context.
- [Adverts Deep Dive](../protocol/adverts-deep-dive.md) — neighbour selection
  for migration.
- [Protocol Design Patterns](protocol-design-patterns.md) — all patterns
  referenced above.
- [Building and Testing](building-and-testing.md) — testing migration handoff
  on a two-node bench without RF.
