# The Dispatcher

`Dispatcher` (`src/Dispatcher.h`, `src/Dispatcher.cpp`) is the lowest layer
in the MeshCore stack that touches the radio. It owns the Rx/Tx loop,
the outbound packet queue, and the duty-cycle budget. Everything above it
(including `Mesh`) inherits from it.

## Role in the stack

```
Radio              ← pure virtual: recvRaw(), startSendRaw(), …
PacketManager      ← pure virtual: allocNew(), queueOutbound(), …
MillisecondClock   ← pure virtual: getMillis()
        │
   Dispatcher      ← cooperative loop(); Rx detection; Tx scheduling
        │
      Mesh         ← payload routing (next layer up)
```

`Dispatcher` is an abstract class — you never instantiate it directly.
`Mesh` (and then your application subclass) provide the concrete
`onRecvPacket()` callback.

## The three pure-virtual dependencies

`Dispatcher`'s constructor takes three interfaces:

| Interface | Responsibility |
|---|---|
| `Radio` | Hardware-agnostic radio: receive, transmit, airtime estimate, CAD |
| `PacketManager` | Packet pool allocation + outbound and inbound priority queues |
| `MillisecondClock` | Monotonic millisecond counter (`millis()` equivalent) |

Keeping these as pure-virtual interfaces means the entire protocol stack
can be unit-tested without real hardware by swapping in stub implementations.

### `Radio`

Key methods your radio driver must implement:

```cpp
int  recvRaw(uint8_t* bytes, int sz);          // poll; returns 0 if nothing
bool startSendRaw(const uint8_t* bytes, int len); // non-blocking TX start
bool isSendComplete();                          // poll TX completion
uint32_t getEstAirtimeFor(int len_bytes);       // airtime estimate (ms)
float packetScore(float snr, int packet_len);   // signal quality score
bool isInRecvMode();                            // is radio in Rx state?
```

RadioLib driver wrappers live in `src/helpers/radiolib/`.

### `PacketManager` / `StaticPoolPacketManager`

The concrete implementation shipped in `src/helpers/StaticPoolPacketManager.h`
maintains three internal `PacketQueue` objects:

- **unused** — free pool (pre-allocated `Packet` objects)
- **send_queue** — outbound packets waiting to be transmitted, ordered by priority and scheduled time
- **rx_queue** — inbound packets received but not yet processed

`allocNew()` pops from the unused pool. `free()` returns a packet there.
`queueOutbound()` pushes to `send_queue` with a priority byte and a scheduled
transmit time. `getNextOutbound(now)` returns the highest-priority packet
whose scheduled time has passed.

The pool is fixed at compile time (configured per-variant via `#define`).
This means no heap allocation at runtime — critical on constrained MCUs.

## What `loop()` does

`Dispatcher::loop()` is called every Arduino `loop()` iteration.
It does two things in order:

### 1. `checkRecv()` — receive path

1. Poll `_radio->recvRaw()`.
2. If bytes arrived, call `tryParsePacket()` to deserialise the wire bytes into a `Packet` struct.
3. Calculate a `packetScore` (SNR + length quality metric).
4. Calculate a **Rx delay**: better-scoring packets are forwarded sooner;
   weaker signals are held longer to let a better copy arrive from another
   direction. Formula: `delay ≈ (10^(0.85 − score) − 1) × airtime`.
5. Call `_mgr->queueInbound(pkt, scheduled_time)` with the computed delay.
6. While the inbound queue has packets whose scheduled time has passed, call
   `processRecvPacket()` → `onRecvPacket()` (the virtual hook `Mesh` overrides).

### 2. `checkSend()` — transmit path

1. Check the **duty-cycle TX budget** (see below). If budget is exhausted, skip.
2. Call `_mgr->getNextOutbound(now)` for the next scheduled packet.
3. If a packet is ready and the radio is idle (not mid-receive or mid-transmit):
   a. Call `_radio->startSendRaw()`.
   b. Track `outbound_start` and `outbound_expiry` to detect hung transmitters.
4. Poll `_radio->isSendComplete()`. On completion, deduct actual airtime
   from `tx_budget_ms`, call `onSendFinished()`, return the radio to Rx mode.

## Duty-cycle budget

`Dispatcher` implements a **token-bucket** airtime limiter:

```
tx_budget_ms  — remaining transmit budget in the current window
duty_cycle    = 1 / (1 + getAirtimeBudgetFactor())
max_budget    = duty_cycle_window_ms × duty_cycle
```

The budget refills continuously at the `duty_cycle` rate (elapsed × duty_cycle
added each `loop()` tick, capped at `max_budget`). Before every transmission
the Dispatcher checks that `tx_budget_ms` is large enough; if not, the packet
stays queued.

`getAirtimeBudgetFactor()` is virtual. `Mesh` subclasses (and `MyMesh`) override
it to implement per-node airtime policies — e.g. a repeater running in a
congested region can throttle itself without any core changes.

The default duty-cycle window is **3 600 000 ms (1 hour)**. Subclasses can
override `getDutyCycleWindowMs()`.

## Packet scoring and Rx delay

When a flood packet is received by multiple nodes, each node independently
decides when to re-transmit. The Rx delay spreads those re-transmissions so
they do not collide:

- High SNR, short packet → low delay → re-transmit quickly
- Low SNR, long packet → high delay → wait for a better copy

This is a core anti-collision mechanism. See
[Routing and Flooding](../protocol/routing-and-flooding.md) for the
protocol-level view.

## CAD (Channel Activity Detection) support

The `Dispatcher` checks whether a CAD-fail event has been signalled by the
radio driver. If a transmission attempt collides with an incoming signal
(channel busy), the packet is re-queued with a random backoff delay:
`getCADFailRetryDelay()` (default 200 ms, overridable). If CAD-fail keeps
happening for longer than `getCADFailMaxDuration()` (default 4 s), the
Dispatcher gives up and releases the packet.

## `DispatcherAction` — what to do with a received packet

`onRecvPacket()` returns a `DispatcherAction` value that tells `Dispatcher`
what to do with the packet after processing:

| Action | Meaning |
|---|---|
| `ACTION_RELEASE` | Done; return packet to pool |
| `ACTION_MANUAL_HOLD` | Application keeps ownership; must release later |
| `ACTION_RETRANSMIT(pri)` | Re-queue for re-transmission at priority `pri` |
| `ACTION_RETRANSMIT_DELAYED(pri, delay)` | Same, but after `delay` ms |

`Mesh::onRecvPacket()` uses these to implement the flood-forward decision.

## Stats tracking

`Dispatcher` maintains four counters (reset via `resetStats()`):

```cpp
n_sent_flood, n_sent_direct   // outbound packets by route type
n_recv_flood, n_recv_direct   // inbound packets by route type
```

plus `_err_flags` for error events (`ERR_EVENT_FULL`, `ERR_EVENT_CAD_TIMEOUT`,
`ERR_EVENT_STARTRX_TIMEOUT`).

## Key takeaways

- **Single-threaded cooperative loop.** No RTOS tasks, no ISR-driven state
  machines in application code. Everything runs in `loop()`.
- **The radio is polled, not interrupt-driven at the application layer.**
  RadioLib handles the low-level ISR; `recvRaw()` surfaces completed packets.
- **Airtime budget is built in.** You do not need separate rate limiting;
  tune `getAirtimeBudgetFactor()` to match your region's duty-cycle rules.
- **All queuing is in the `PacketManager`.** Swap in a different implementation
  to get different scheduling behaviour without touching the protocol logic.
