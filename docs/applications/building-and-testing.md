# Building and Testing

From working protocol design to a good mesh citizen: the development loop,
testing without live RF, debugging techniques, and versioning discipline.

## Choosing your implementation surface

Before you start coding, commit to one of:

| Surface | When it fits | Trade-offs |
|---|---|---|
| **Host-side via Companion API** | Logic runs on a capable host (Pi, laptop, phone); node is a radio modem | Fast iteration; no firmware toolchain; limited to what the Companion API exposes |
| **Firmware-side custom payload type** | Logic must run on the node MCU; no host attached; relay/aggregation in firmware | Full access to mesh internals; requires PlatformIO build; slower iteration |

Start host-side. Move to firmware only when you have a concrete reason: the
application needs to run headless in the field, or it needs access to internals
(routing tables, ACK callbacks) that the Companion API does not expose.

For firmware-side work, see [Custom Application](../extending/custom-application.md)
and [Build from Source](../extending/build-from-source.md) in Extending MeshCore.

## The host-side development loop

```
1. Pick a language with serial/BLE support (Python, Rust, Go, TypeScript…)
2. Open a serial connection to a Companion Radio node
3. Send a LOGIN frame, get OK
4. Send your application frames; inspect responses
5. Iterate on protocol design in code, not on the air
```

The `examples/companion_radio` firmware is the correct firmware to flash on
your development node. Its `DataStore` and `MyMesh` classes show the host
frame model in full. The Companion API frame catalogue is at
[docs.meshcore.io/companion_protocol](https://docs.meshcore.io/companion_protocol/).

For Python, the `meshcore-ha` project (`~/Documents/code/meshcore-ha`) is a
reference host-side implementation that speaks the Companion API — it is worth
reading before writing your own client from scratch.

## Testing without RF

You do not need radio range to test your protocol. Two approaches:

### Two nodes on a bench

Flash two Companion Radio nodes. Place them within a few centimetres of each
other. LoRa at SF10/BW250 with 20 dBm TX power will communicate at zero
distance with no additional attenuation. Connect both to your dev machine via
USB. Run your sender and receiver host apps against the two nodes.

**Tip:** reduce TX power to 2–5 dBm for bench testing to avoid saturating the
receiver with a signal that is too strong (some LoRa chips behave poorly at
very high RSSI).

### Loopback via KISS modem

The `examples/kiss_modem` firmware exposes raw LoRa packet access via KISS
framing over serial. You can inject and capture raw packets from a host
without a second radio, useful for unit-testing packet parsers. See
[KISS and Raw Packets](../extending/kiss-and-raw-packets.md) for the setup.

### Unit-testing your packet codec

Write unit tests for your encode/decode functions completely offline. The
packet format is deterministic; there is no network dependency. Test:

- That `encode(decode(encode(msg))) == encode(msg)`.
- That a type byte of `0x00` (reserved) is rejected.
- That a packet truncated at every possible byte raises a clear error.
- That duplicate `msg_id` / `cmd_id` / `req_id` values are handled correctly.
- That a version byte higher than your max-supported version returns a graceful
  error, not a panic.

## Debugging on the mesh

### RSSI and path inspection

The Companion API returns RSSI and SNR in receive frames. Log them. Unexpectedly
low RSSI explains delivery failures that look like protocol bugs.

The trace mechanism (`PAYLOAD_TYPE_TRACE`) records per-hop SNR as a packet
traverses the mesh. Use it to understand multi-hop path quality before blaming
your application protocol. See
[Packet Journey](../protocol/packet-journey.md) for the trace conceptual model.

### Capturing raw traffic with KISS

Flash one node with `kiss_modem` as a passive sniffer (no TX). Capture all
LoRa frames in range and dump to a log. Compare what you transmitted to what
was captured over the air. This catches encoding errors that look like
delivery failures.

### Logging `req_id` / `cmd_id` lifecycles

Log every outgoing request ID at creation, every incoming response ID at
receipt, and every retry. The log should show:

```
2026-06-06T14:00:00Z send MSG_SEND req_id=0xAB12CD34 to=0x7F3A...
2026-06-06T14:00:45Z no response, retry 1 req_id=0xAB12CD34
2026-06-06T14:01:45Z recv MSG_ACK req_id=0xAB12CD34 status=OK
```

If you see a `resp_id` that does not match any outstanding request, you have a
correlation bug. If you never see a response, it is a delivery or routing issue.

## Airtime etiquette on a shared mesh

Your application shares the mesh with other users. Obligations:

1. **Honour the `airtime_factor`** configured on your Companion Radio node.
   If your host app sends faster than the node can forward, packets are dropped
   at the node — add a transmit queue with backpressure.
2. **Back off exponentially on retry** — do not retry at a fixed short interval.
3. **Suppress unnecessary traffic.** Do not send a heartbeat/keepalive if the
   mesh layer's adverts already serve that purpose.
4. **Test your worst-case transmit rate** before deploying. Simulate the
   maximum number of nodes your application will have and calculate airtime
   consumed per minute. If it exceeds ~10 % of available channel capacity,
   redesign the protocol.
5. **Do not use flood mode for unicast traffic.** Use direct routing
   (`sendMessage` / `REQ`) when you have a known path to the destination.
   Flood consumes channel capacity proportional to the number of nodes in
   range.

## Versioning and forward compatibility

Once you have deployed nodes in the field, you cannot atomically upgrade all of
them. Design for **mixed-version operation** from day one:

- Include a `version` byte in every message (see
  [Protocol Versioning](protocol-design-patterns.md#protocol-versioning)).
- Parse fields you know; skip fields you do not recognise.
- Never change the meaning of existing fields; add new fields at the end.
- Keep a compatibility matrix: `min_version` your sender requires in receivers,
  and `min_version` your receiver accepts from senders.
- For firmware-side payload types: register your payload type number before
  deploying any nodes (see
  [Number Allocations](https://docs.meshcore.io/number_allocations/)).

## From prototype to production checklist

- [ ] Protocol version byte is present in all messages.
- [ ] All message types have explicit handling for unknown `type` bytes
      (log and ignore, not crash).
- [ ] Retry logic uses exponential backoff with a cap.
- [ ] Idempotency guard is in place (duplicate `req_id` / `cmd_id` returns
      cached response).
- [ ] Airtime budget is calculated for the maximum expected node count.
- [ ] Unit tests cover encode/decode round-trips for every message type.
- [ ] Two-node bench test has been run end-to-end.
- [ ] RSSI logging is present in the receiver for field debugging.
- [ ] A `GAME_SYNC_REQ` / resync path (or equivalent state-repair mechanism)
      is implemented and tested.
- [ ] Behaviour when the destination is unreachable for 1 hour is defined and
      tested.

---

## Where next in your journey

**Host-side work is done — what if you need more?**

Most applications ship entirely on the host side and never need to touch
firmware. If you have reached the limits of what the Companion API exposes —
routing tables, ACK callbacks, on-device processing, new sensor hardware —
the door to the firmware side is:

[Extending MeshCore →](../extending/index.md) — custom sensors, custom payload
types, custom board variants, and contributing upstream.

**For quick lookups:**

- [Glossary](../reference/glossary.md) — every term used in the guide
- [Spec Index](../reference/spec-index.md) — all `docs.meshcore.io` pages with
  one-liners
- [Regions and Frequencies](../reference/regions-and-frequencies.md) — RF
  presets and duty-cycle rules by region
