# Worked Example: Multiplayer Game

**Patterns used:**
[message framing and type bytes](protocol-design-patterns.md#message-framing-and-type-bytes),
[request/response correlation](protocol-design-patterns.md#requestresponse-correlation),
[sequencing and ordering](protocol-design-patterns.md#sequencing-and-ordering),
[discovery and announce](protocol-design-patterns.md#discovery-and-announce),
[store-and-forward](protocol-design-patterns.md#store-and-forward),
[protocol versioning](protocol-design-patterns.md#protocol-versioning)

## What we're building

A **turn-based** multiplayer game where 2–4 players within mesh radio range
take turns, with game state replicated across all participants. Think chess,
word games, or simple strategy games — not anything requiring sub-second
reactions.

This example is deliberately honest about what the mesh cannot support, and
shows how to design around those limits rather than pretend they don't exist.

## Why real-time is impractical

Before designing the game, understand the channel:

- **Round-trip latency** for a 2-hop path is roughly 10–30 seconds including
  retransmit windows.
- **Throughput** is on the order of 1–2 small packets per minute per node if
  you want to stay within duty-cycle limits.
- **Packet loss** of 10–20 % per hop is common in challenging RF environments.

A real-time game (racing, shooter, etc.) requires sub-100 ms update rates.
That is physically impossible on LoRa. Turn-based games, where each player
acts once per "round" and there is no global clock, are a natural fit.

The mesh is an excellent transport for:
- Asynchronous board games (chess, Go, word games)
- Text-based adventure games
- Trivia or quiz games
- Simple strategy games with 30-second turn timers

## Architecture

All game traffic uses a **private channel** (pre-shared key) so the game is
isolated from the general mesh population. Players discover each other via
advert scanning and join a lobby before the game starts.

```
Player A host app                Player B host app
    │ Companion API                  │ Companion API
    ▼                                ▼
Companion Radio A ──[LoRa]──► Companion Radio B
        │                               │
        └────────[LoRa]─────────────────┘
                (flood on game channel)
```

For 2–4 players all within direct radio range, a **flood on the game channel**
is simplest: every game packet reaches every player. With PAYLOAD_TYPE_GRP_DATA,
all players receive each move; no unicast routing needed.

## Lobby and discovery

Before a game starts, players enter a **lobby**:

1. Each player's host app broadcasts a `GAME_LOBBY_ANNOUNCE` on the game
   channel at 60-second intervals (see
   [Discovery and Announce](protocol-design-patterns.md#discovery-and-announce)).
2. Players with compatible game versions see each other's announces and can
   invite or accept invitations.
3. One player becomes the **host** (the first to send `GAME_START`). The host's
   public key prefix is the game session ID.

```
┌─────────┬──────┬────────────────────┬────────┬──────────────────────┐
│ version │ type │ game_id (7 B)      │ seats  │ display name (≤32 B) │
│  1 B    │  1 B │ (host key prefix)  │  1 B   │                      │
└─────────┴──────┴────────────────────┴────────┴──────────────────────┘
```

`seats` = maximum number of players. Players join by sending `GAME_JOIN`
with the same `game_id`. The host collects joins until `seats` is full, then
broadcasts `GAME_START`.

## Game state model

The host **owns the authoritative game state**. On each turn:

1. The current player broadcasts their move as `GAME_MOVE`.
2. The host validates the move, applies it to game state, and broadcasts
   `GAME_STATE` (a compact snapshot of the full game state).
3. All players update their local view from `GAME_STATE`.

Why broadcast `GAME_STATE` rather than just `GAME_MOVE`? Because packets are
lost. A player who missed the last move would need to request a resync anyway;
including the full state in every turn message makes the protocol **loss-tolerant
by design** — any player can reconstruct current state from the latest `GAME_STATE`,
regardless of how many moves they missed.

### Message types

| Type byte | Name | Payload |
|---|---|---|
| `0x01` | `GAME_LOBBY_ANNOUNCE` | game_id, seats, name |
| `0x02` | `GAME_JOIN` | game_id, player name (≤16 B) |
| `0x03` | `GAME_START` | game_id, player list (up to 4 × 7+16 B) |
| `0x04` | `GAME_MOVE` | game_id, move_seq (2 B), move payload (variable) |
| `0x05` | `GAME_STATE` | game_id, move_seq (2 B), state snapshot |
| `0x06` | `GAME_OVER` | game_id, winner_key (7 B), reason (1 B) |
| `0x07` | `GAME_SYNC_REQ` | game_id, last_seen_seq (2 B) |
| `0x08` | `GAME_SYNC_RESP` | game_id, full state snapshot |

### Turn sequencing

Each `GAME_MOVE` and `GAME_STATE` carries a `move_seq` — a monotonically
increasing 16-bit counter, incremented by the host on each accepted move. Players
track the last `move_seq` they have applied. If a player receives a `GAME_STATE`
with `move_seq` higher than expected-next, they are out of sync and broadcast
`GAME_SYNC_REQ` with their last-seen `move_seq`. The host responds with
`GAME_SYNC_RESP` containing the full current state.

This is the [sequencing and ordering pattern](protocol-design-patterns.md#sequencing-and-ordering)
adapted for game state: rather than buffering individual moves, resync from
the canonical state snapshot.

## State snapshot sizing

The full game state must fit in a `GAME_STATE` packet. Available payload:

- GRP_DATA overhead: ~19 bytes
- Envelope (version + type + game_id + move_seq): ~11 bytes
- Available for state: ~154 bytes

For a chess-like game, an 8×8 board with 12 piece types fits in 32 bytes using
4 bits per square. A word game with a 15×15 board needs 113 bytes for the
board (using one ASCII byte per square, with 0x00 for empty). Most simple turn-
based games fit comfortably.

For larger state, use the [fragmentation pattern](protocol-design-patterns.md#fragmentation-and-reassembly)
with a `game_id + move_seq + frag` header.

## Trust and anti-cheat notes

In a small group of known players (friends using a shared mesh), trust is
implicit. In a wider-deployed game:

- **All players receive all moves** (flood channel), so cheating (sending an
  invalid move) is immediately visible to all players.
- **The host validates moves.** If the host rejects a move, it broadcasts
  `GAME_STATE` unchanged. The cheating player's move does not advance the game.
- **The host role is not Byzantine-fault-tolerant.** A cheating host can
  produce an invalid `GAME_STATE`. Mitigations:
  - Rotate the host role across players (host = player whose turn it is).
  - Have all players independently validate `GAME_STATE` against their copy of
    the rules; disconnect from any host whose state fails validation.
- **There is no server.** Unlike a networked game with a dedicated server, the
  mesh game relies on in-network trust. Design your game for groups where
  social trust is sufficient.

## Duty-cycle accounting for a game

A 4-player chess game with 1-minute turns generates:

| Event | Packets | Frequency |
|---|---|---|
| `GAME_MOVE` | 1 per player turn | 1/minute (if each player moves once/min) |
| `GAME_STATE` | 1 per player turn | 1/minute |
| `GAME_LOBBY_ANNOUNCE` | 1 per player | 1/minute (pre-game only) |

Peak: ~2 packets/minute = ~1 packet/30 s. At 600 ms air time per packet, that
is 2 % of a 60-second window — comfortably within EU868 1 % duty cycle per
sub-band, assuming channel spacing is configured correctly.

A 30-second turn timer triples this to ~6 % — still legal. Do not go faster
than 30-second turns without checking your regional limits.

## What the mesh cannot support

Be honest with your users:

- **No real-time**: turns must be 30 seconds minimum for comfortable play.
- **No hidden information**: all packets are broadcast to all players on the
  channel; a player who captures LoRa traffic can read every packet (even if
  it is encrypted with the channel PSK, anyone with the PSK can read it).
  Design games where hidden information is not a gameplay requirement, or
  implement an out-of-band secret-sharing protocol using unicast REQ/RESPONSE.
- **No guaranteed simultaneity**: two players taking a "simultaneous" action
  (like real-time bidding) will have their packets arrive in an undefined order.
  Design for serial turns only.
- **Nodes can drop out**: if a player's node goes offline mid-game, the game
  stalls unless you implement a timeout-and-skip mechanism.

## Cross-links

- [Companion API](../companion-api/index.md) — sending GRP_DATA from the
  host app.
- [Channels vs. Direct](../concepts/channels-vs-direct.md) — why channel
  flooding is right for this use case.
- [Payload Types Tour](../protocol/payload-types-tour.md) — GRP_DATA in context.
- [Protocol Design Patterns](protocol-design-patterns.md) — all patterns
  referenced above.
- [Building and Testing](building-and-testing.md) — simulating multiple
  players on a bench.
