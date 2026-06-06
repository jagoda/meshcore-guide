# Regions and Frequencies

MeshCore operates in unlicensed sub-GHz ISM bands. The correct frequency plan depends on your country's regulations and the hardware you are using. This page summarises the main regions, common community presets, and the rules that apply to each.

For the canonical list of region codes and frequency-preset values used in `number_allocations.md` and `RegionMap`, see the [Number Allocations spec](https://docs.meshcore.io/number_allocations/) on `docs.meshcore.io`.

---

## Regional frequency plans

| Region | Band | Typical centre frequency | Duty-cycle limit | Notes |
|--------|------|--------------------------|-----------------|-------|
| **EU868** | 863–870 MHz | 869.525 MHz | 1% (≈36 s/hr) | Europe, UK, most of ETSI territory |
| **US915** | 902–928 MHz | 910.525 MHz | No hard cap (FCC Part 15) | USA, Canada, much of the Americas |
| **AU915** | 915–928 MHz | 915.2 MHz | No hard cap (AS/NZS) | Australia, New Zealand |
| **AS433** | 433.05–434.79 MHz | 433.175 MHz | Varies by country | Some Asian markets; check local rules |
| **IN865** | 865–867 MHz | 865.0625 MHz | 1% | India |
| **KR920** | 920–923 MHz | 921.9 MHz | 1% | South Korea |
| **CN470** | 470–510 MHz | 470.3 MHz | No hard cap | China |

!!! warning "Always use the correct region for your hardware"
    LoRa radio modules and antennas are tuned for a specific frequency range. Using a 915 MHz module on an 868 MHz frequency plan (or vice versa) degrades performance and may violate regulations. Flash the region-appropriate firmware preset from [meshcore.io/flasher](https://meshcore.io/flasher).

!!! warning "Duty cycle is a legal requirement"
    In EU868 and similar regulated bands, the 1% duty cycle cap is not advisory — it is a legal obligation. A radio that transmits for 1 second must wait 99 seconds before transmitting again on the same channel. MeshCore's architecture (companions don't repeat, direct routing reduces floods) keeps usage well within this limit under normal operation.

---

## LoRa modulation parameters

All nodes on the same sub-mesh must use **identical** modulation parameters or they cannot hear each other.

| Parameter | Abbreviation | Common values | Effect on airtime |
|-----------|-------------|---------------|------------------|
| Centre frequency | — | See table above | Must match exactly |
| Bandwidth | BW | 62.5, 125, 250 kHz | Narrower ↑ sensitivity, ↑ airtime |
| Spreading Factor | SF | SF7 – SF12 | Higher SF ↑ range, ↑↑ airtime |
| Coding Rate | CR | 4/5, 4/6, 4/7, 4/8 | Higher CR ↑ error-correction, ↑ airtime slightly |

### How SF affects airtime (approximate, 50-byte payload)

| SF | BW | Approx. airtime |
|----|----|----------------|
| SF7 | 125 kHz | ~60 ms |
| SF9 | 125 kHz | ~250 ms |
| SF11 | 125 kHz | ~1 s |
| SF12 | 125 kHz | ~3–4 s |
| SF7 | 62.5 kHz | ~120 ms |

High airtime reduces throughput for everyone sharing the channel, burns more duty-cycle budget, and increases collision probability. Prefer the lowest SF that gives reliable coverage.

---

## Community consensus presets (as of 2025)

Regional MeshCore communities have converged on specific settings after real-world testing. These are the most common; check your regional community on [Discord](https://discord.gg/meshcore) or the community forum for current recommendations.

### EU868 community preset
- **Frequency:** 869.525 MHz
- **SF:** SF9, BW125 (common), or SF7/BW62.5 (active urban meshes)
- **Notes:** 869.525 MHz sits in the EU SRD860 band with a 10% duty-cycle allowance (higher than the standard 1%), making it popular for mesh use.

### US915 / AU915 community preset
- **Frequency:** 910.525 MHz
- **SF:** SF7, BW62.5
- **CR:** CR5 (4/5)
- **Notes:** Narrowband BW62.5 at SF7 gives sub-200 ms airtime per packet and good sensitivity. Community default before the narrowband shift was SF9/BW125.

### Changing your radio parameters

```bash
# Via USB serial admin CLI:
set freq 910.525
set sf 7
set bw 62
set cr 5
```

Or use the companion app's **Settings → RF Settings → Region preset** and select the preset that matches your region and community.

---

## Multi-region bridging

MeshCore supports **transport repeaters** that bridge two independent RF regions (e.g., an EU868 cluster and a US915 cluster via IP backhaul). Transport repeaters use `ROUTE_TYPE_TRANSPORT_FLOOD` and `ROUTE_TYPE_TRANSPORT_DIRECT` route types with a 4-byte transport-code block derived from the `RegionMap`.

The `RegionMap` helper (`src/helpers/RegionMap.*`) maintains a named tree of regions and the transport keys used to authenticate cross-region packet forwarding. Region configuration is done via the admin CLI.

Multi-region bridging is an advanced feature; see the [Number Allocations spec](https://docs.meshcore.io/number_allocations/) for region-code assignments and the `RegionMap` source for the data model.

---

## Regional community resources

Regional MeshCore communities maintain their own getting-started guides and
consensus frequency tables. These are the right place for deployment specifics
(antenna heights, local interference sources, link-budget measurements) that
are too regional to belong in a general guide:

- **[MeshCore Discord](https://meshcore.gg)** — find your regional channel under `#regions`. This is the primary place to ask "what settings does my local mesh use?"
- **[map.meshcore.io](https://map.meshcore.io)** — see live nodes near you and infer which frequency plan your region's mesh is already using before you transmit.
- **[MeshCore FAQ §Regions](https://docs.meshcore.io/faq/)** — common regional setup questions.

For North American and European community-specific guides, check the Discord for pinned links — some regions have published their own setup walkthroughs.

---

## See also

- [Airtime and Regions](../concepts/airtime-and-regions.md) — conceptual guide to airtime, duty cycle, and why they matter.
- [Number Allocations spec](https://docs.meshcore.io/number_allocations/) — canonical region codes and frequency-preset values.
- [Flash Your First Device](../getting-started/flash-your-first-device.md) — selecting the correct region preset during flashing.
- [Running a Repeater](../operating/running-a-repeater.md) — changing RF parameters on a deployed repeater.
