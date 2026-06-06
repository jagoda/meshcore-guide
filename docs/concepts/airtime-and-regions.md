# Airtime and Regions

LoRa radios share unlicensed spectrum with millions of other devices. Regional
regulators set strict rules on how much of the channel any one device may
occupy. Understanding **airtime** and **duty cycle** is not just good practice —
in most jurisdictions it is a legal requirement.

---

## The ISM bands MeshCore uses

MeshCore operates in the **sub-GHz ISM (Industrial, Scientific, Medical)**
bands:

| Region | Frequency band | Common examples |
|--------|---------------|-----------------|
| Europe / UK | 863–870 MHz (EU868) | Germany, France, UK, most of Europe |
| USA / Canada | 902–928 MHz (US915) | Also Australia, New Zealand, and much of the Americas |
| Asia | 433 MHz | Some markets; check local rules |

!!! warning "Match your hardware to your region"
    LoRa antennas and radio modules are tuned for a frequency range. Using a
    915 MHz module in an 868 MHz region (or vice versa) will degrade
    performance and may violate regulations. Check [flasher.meshcore.io](https://meshcore.io/flasher) for region-appropriate firmware presets.

---

## What is airtime?

**Airtime** is the duration a radio occupies the channel during a single
transmission. It is determined by three LoRa parameters:

- **BW — Bandwidth:** the slice of spectrum used. Narrower bandwidth = longer
  airtime but better sensitivity. Common values: BW62.5, BW125, BW250 kHz.
- **SF — Spreading Factor:** how much each symbol is spread in time. Higher SF
  = longer range but dramatically more airtime. Each SF step roughly doubles
  transmission time.
- **CR — Coding Rate:** the proportion of redundancy bits added for error
  correction (4/5, 4/6, 4/7, 4/8). Higher CR = slightly longer airtime, better
  recovery from interference.

A packet sent at SF12/BW125 can take **3–4 seconds on air**. The same packet at
SF7/BW125 takes under 100 ms. Spreading factor has the largest single impact on
airtime.

!!! tip "MeshCore community recommendations (as of 2025)"
    Many regions have moved to narrower bandwidth settings — BW62.5 with SF7,
    SF8, or SF9 — after testing showed better SNR, lower noise floor, and
    faster transmissions in the real ISM band environment. Example:
    USA/Canada preset is 910.525 MHz, SF7, BW62.5, CR5. Check your regional
    community on [MeshCore Discord](https://meshcore.gg) for current consensus.

---

## Duty cycle: the hard limit

Most regional regulations cap the fraction of time a radio may transmit. In
EU868 the typical limit is **1% duty cycle** — a radio may transmit for at
most 36 seconds in any rolling hour.

Exceeding the duty cycle:

- Interferes with other users of the shared band.
- Is illegal in most jurisdictions.
- Can cause your radio to be silenced by firmware duty-cycle enforcement.

MeshCore's architecture naturally limits airtime:

- **Companion radios do not repeat.** Only designated repeaters retransmit
  packets, keeping each node's transmission count low.
- **Direct routing after path discovery.** Only 2–4 nodes transmit a
  direct-routed message vs. every repeater on the mesh for a flood.
- **Advert intervals are configurable.** The repeater flood-advert interval
  (default 12 h) keeps beacon traffic minimal.

---

## Setting the right frequency for your region

The frequency is set once at device setup:

- **Via USB serial / web config:** `set freq <MHz>` (e.g., `set freq 868.0`)
- **Via companion app:** Settings → RF Settings → Region preset

Choose a preset that matches your country's regulations. If your community has
agreed on a specific narrowband setting, use the same BW and SF values so your
device is on the same channel as your peers.

!!! note "All nodes on the same sub-mesh must use the same RF parameters"
    BW, SF, CR, and frequency must match exactly for two nodes to hear each
    other. A device on SF7/BW62.5 cannot hear a device on SF11/BW125, even
    at the same centre frequency.

---

## What happens when duty cycle is exceeded

In EU868, the radio module typically enforces duty cycle in firmware. If you
exceed the limit:

1. The radio refuses to transmit and returns an error.
2. The MeshCore firmware queues the packet.
3. Once the duty-cycle window resets, the packet transmits.

In regions with less stringent enforcement (US915), duty cycle is not legally
capped in the same way, but **best practice** is still to minimise unnecessary
transmissions — channel noise affects everyone sharing the band.

---

## Airtime and your repeater strategy

Every repeater retransmit consumes airtime from every radio in range. Poorly
placed repeaters — especially ones receiving the same packet on multiple paths —
create a **collision storm** that degrades the entire local mesh.

Good practices:

- Place repeaters at elevation with a clear RF horizon, not in valleys where
  they hear too many nodes.
- Tune `set flood.max <hops>` to limit how far channel floods propagate.
- Use direct messaging (path-routed) wherever possible to minimise flood traffic.
- Monitor airtime with an Observer node if you need metrics.

---

## What's next

- [Nodes and Roles](nodes-and-roles.md) — which node types repeat and which
  stay silent.
- [How Messages Travel](how-messages-travel.md) — how direct routing reduces
  channel airtime compared to flooding.
- [Number Allocations spec](https://docs.meshcore.io/number_allocations/) —
  the canonical list of frequency presets and region codes.
