# Power and Battery

Always-on infrastructure — repeaters on rooftops, sensors in fields, room servers in garages — runs from battery, solar, or a combination. This page covers the practical operator side: sizing, charging topologies, duty-cycle tuning, and the nRF52 power management features available on supported boards.

For byte-level nRF52 power management internals (LPCOMP configuration, register layout, board implementation), see the [nRF52 Power Management spec](https://docs.meshcore.nz/nrf52_power_management/) on docs.meshcore.nz.

---

## Power budget fundamentals

A MeshCore repeater is mostly **listening** (RX) with occasional **transmitting** (TX). Typical power profiles:

| State | Typical current draw | Notes |
|-------|---------------------|-------|
| RX (listening) | 5–15 mA | Varies by board and radio chip |
| TX (LoRa burst) | 80–120 mA | Depends on power setting and PA stage |
| Sleep (nRF52 boards) | < 0.05 mA | With power-saving enabled |
| ESP32 light sleep | 2–5 mA | Active radio required for RX |

**Rule of thumb for always-on repeater (no sleep, nRF52):**

- Average current ≈ RX current + (TX duty fraction × TX current)
- At 50% duty cycle and 10 mA RX / 100 mA TX: ~55 mA average
- For 24-hour operation: ~1.3 Ah per day
- A 3.7 V, 5000 mAh LiPo gives roughly 3–4 days of runtime

Actual numbers depend heavily on your board, antenna, radio settings, and how many packets the repeater forwards.

---

## Battery topology

### LiPo / Li-Ion

Most MeshCore-supported boards include a JST connector and an onboard charger. A single-cell 3.7 V LiPo is the standard choice.

- **Capacity:** 2000–10000 mAh for multi-day standalone operation.
- **Charging:** Boards with onboard USB charging (TP4054, MCP73831, etc.) can be solar-charged via a cheap USB solar panel.
- **Voltage range:** Nominal 3.7 V; full charge 4.2 V; cutoff (to protect the cell) ~3.0–3.3 V.

### Solar charging

Pair with a **solar charge controller** (MPPT or PWM) sized to your panel:

- Small panel (0.5–1 W, 5 V USB): suitable for nRF52 boards with power-saving enabled in low-traffic networks.
- Medium panel (2–5 W): comfortable margin for an always-on ESP32 repeater with a 3000 mAh battery.

Place panels with maximum daily sun exposure; even a little shading dramatically reduces output.

### USB / mains power

For indoor or sheltered deployments, a standard USB power supply is simplest. Use a quality supply — cheap ones introduce RF noise that degrades receive sensitivity.

---

## Firmware-side power controls

### Duty cycle limiting

Every MeshCore infrastructure node can cap the fraction of time its transmitter is on:

```
get dutycycle           — view current limit (%)
set dutycycle 10        — limit to 10% (EU 868 MHz legal requirement)
set dutycycle 50        — default (50%)
set dutycycle 100       — no limit (not recommended in regulated bands)
```

After each transmission the firmware enforces a quiet period proportional to the on-air time before it transmits again. This is both a regulatory compliance tool (EU sub-GHz duty cycle rules) and a battery conservation measure.

The older `set af <value>` (airtime factor) command is deprecated as of firmware v1.15.0 — use `set dutycycle` instead.

### Power-saving mode (repeater firmware)

On supported boards, enabling power-saving puts the microcontroller into sleep between radio events:

```
powersaving on      — enable sleep between transmissions
powersaving off     — always-on (default)
powersaving         — query current state
```

This is most effective on nRF52-based boards where sleep current can drop below 50 µA. Not all boards implement deep sleep for the radio — check your board's documentation.

### Flood advert interval

The periodic flood advert is a scheduled transmission. Lengthening the interval reduces transmit events:

```
set flood.advert.interval 24    — advert once per day (saves airtime vs default 12 h)
```

For solar+battery deployments where the repeater may sleep through the night, a 24-hour interval is reasonable.

---

## nRF52 power management (hardware-level)

nRF52-based boards (RAK4631, Heltec T114, XIAO nRF52840, SenseCAP T1000-E, GAT562 Mesh Watch13) have hardware-level power management built into the MeshCore firmware. Key features:

### Boot voltage lockout

Before starting mesh operations, the firmware checks the battery voltage. If it is below the configured threshold (e.g. 3300 mV), the node enters **protective shutdown** (SYSTEMOFF) instead of booting — preventing brownouts and flash corruption from repeated failed boot loops at critically low charge.

This lockout is skipped when USB power (VBUS) is detected.

### LPCOMP voltage wake

When the node enters SYSTEMOFF due to low voltage, the **Low Power Comparator** (LPCOMP) is armed. It monitors the battery voltage and automatically wakes the node when the voltage rises above the recovery threshold (e.g. after solar charging resumes). VBUS detection also wakes the node if USB power is connected.

### Boot reason CLI queries

After any boot you can query why the node last restarted:

```
get pwrmgt.support      — "supported" or "unsupported"
get pwrmgt.source       — "battery" or "external" (USB power)
get pwrmgt.bootreason   — e.g. "Wake from LPCOMP / Low Voltage"
get pwrmgt.bootmv       — battery voltage at boot time (mV)
```

These are invaluable for remote diagnostics: if a repeater went offline overnight, `get pwrmgt.bootreason` tells you whether it was a low-voltage shutdown or something else.

### Supported boards (Phase 1)

Boards with boot-lockout + LPCOMP wake + shutdown reason tracking as of the current firmware:

| Board | Implemented |
|-------|------------|
| XIAO nRF52840 | Yes |
| RAK4631 | Yes |
| Heltec T114 | Yes |
| GAT562 Mesh Watch13 | Yes |
| SenseCAP Solar | Yes |

Other nRF52 boards (T1000-E, Nano G2 Ultra, WIO Tracker, etc.) do not yet implement Phase 1. See the [nRF52 Power Management spec](https://docs.meshcore.nz/nrf52_power_management/) for the full board table and Phase 2 roadmap (runtime voltage monitoring, load shedding).

---

## Practical deployment checklist

- [ ] Battery capacity ≥ 3 days of estimated consumption (buffer for cloudy days if solar).
- [ ] Solar panel rated ≥ 2× daily consumption (accounting for panel efficiency and partial shading).
- [ ] `set dutycycle` matches your regional regulatory requirement.
- [ ] `powersaving on` evaluated for your board — check if sleep is implemented.
- [ ] `set flood.advert.interval` set to the longest interval that still meets your network's re-discovery needs.
- [ ] `get pwrmgt.bootmv` checked after first outdoor deployment to confirm the boot voltage is healthy.
- [ ] Enclosure is weatherproof; cable entries are sealed.

---

## Reading battery voltage remotely

For boards with ADC battery sensing, the repeater stats include battery voltage:

```
stats-core      — shows batt_milli_volts field (serial only)
```

Or via nRF52 power management:

```
get pwrmgt.bootmv   — voltage at last boot (not current; updates each reboot)
```

For real-time current voltage, query `stats-core` over serial or from the companion admin CLI stats tab.

If the reported voltage is wrong for your board, calibrate with:

```
get adc.multiplier      — current multiplier (0.0 = use board default)
set adc.multiplier 2.1  — adjust until reading matches a multimeter
```
