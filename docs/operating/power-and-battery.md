# Power and Battery

Always-on infrastructure — repeaters on rooftops, sensors in fields, room servers in garages — runs from battery, solar, or a combination. This page covers the practical operator side: sizing, charging topologies, duty-cycle tuning, and the nRF52 power management features available on supported boards.

For byte-level nRF52 power management internals (LPCOMP configuration, register layout, board implementation), see the [nRF52 Power Management spec](https://docs.meshcore.io/nrf52_power_management/) on docs.meshcore.io.

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
- **Voltage range:** Nominal 3.7 V; full charge 4.2 V; cutoff (to protect the cell) ~3.0–3.3 V. On the handful of nRF52 boards that still ship the firmware boot lockout, the node refuses to boot below 3100 mV — see [Boot voltage lockout](#boot-voltage-lockout).

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
set flood.advert.interval 24    — advert once per day (saves airtime vs default 47 h)
```

For solar+battery deployments where the repeater may sleep through the night, a 24-hour interval is reasonable.

---

## nRF52 power management (hardware-level)

Some nRF52 boards have hardware-level power management built into the MeshCore firmware. It is not universal: the code is compiled in only when a board variant sets the `NRF52_POWER_MANAGEMENT` build flag, and **as of v1.18.0 only three boards do** (see [Boards with power management](#boards-with-power-management-at-v1180)). Key features:

### Boot voltage lockout

Before starting mesh operations, a board with power management compiled in reads the battery voltage. If it is below that board's `PWRMGT_VOLTAGE_BOOTLOCK` threshold, the node enters **protective shutdown** (SYSTEMOFF) instead of booting — preventing brownouts and flash corruption from repeated failed boot loops at critically low charge.

The threshold is per-board, set in the variant's `variant.h`. It is **not** a firmware-wide 3300 mV: on all three boards that still ship the lockout at v1.18.0 it is **3100 mV**. A reading at or below 1000 mV is treated as an ADC glitch and does not trigger a shutdown.

This lockout is skipped when USB power (VBUS) is detected, so a flat-but-plugged-in node boots normally.

!!! note "v1.18.0: the lockout survives on three boards only"
    Until v1.17.1 thirteen nRF52 variants compiled this in. At v1.18.0
    `NRF52_POWER_MANAGEMENT` is set for **Heltec T1**, **Heltec Tower V2**, and
    **Muzi Base** only. Everything else has no boot lockout: the node boots
    regardless of battery voltage. On firmware v1.17.1 and earlier, assume any
    nRF52 board you did not flash yourself may still refuse to boot on a low
    battery.

### LPCOMP voltage wake

When the node enters SYSTEMOFF because of the boot lockout, the **Low Power Comparator** (LPCOMP) is armed. It monitors the battery voltage and automatically wakes the node when the voltage rises above the recovery threshold (e.g. after solar charging resumes). VBUS detection also wakes the node if USB power is connected. Boards without power management compiled in get neither behaviour.

### Boot reason CLI queries

After any boot you can query why the node last restarted:

```
get pwrmgt.support      — "supported" or "unsupported"
get pwrmgt.source       — "battery" or "external" (USB power)
get pwrmgt.bootreason   — e.g. "> Reset: Wake from LPCOMP; Shutdown: Boot Protection"
get pwrmgt.bootmv       — battery voltage at boot time (mV)
```

These are invaluable for remote diagnostics: if a repeater went offline overnight, `get pwrmgt.bootreason` tells you whether it was a boot-protection shutdown or something else.

Only `get pwrmgt.bootreason` works without power management — it answers `> Reset: Not available; Shutdown: Not available` there. `get pwrmgt.support` answers `unsupported`, and `get pwrmgt.source` / `get pwrmgt.bootmv` return `ERROR: Power management not supported`.

### Boards with power management at v1.18.0

Composition as of firmware **v1.18.0** (verified against the variant build flags and `variant.h` thresholds at tag `repeater-v1.18.0`):

| Board | Variant | Boot lockout | LPCOMP wake |
|-------|---------|--------------|-------------|
| Heltec T1 | `heltec_t1` | 3100 mV | Yes |
| Heltec Tower V2 | `heltec_tower_v2` | 3100 mV | Yes |
| Muzi Base | `muzi_base` | 3100 mV | Yes |

Every other nRF52 board — RAK4631, RAK3401, Xiao nRF52840, Heltec T114, Heltec T096, GAT562 Mesh Watch13 / Tracker Pro / EVB Pro / 30S Mesh Kit, SenseCAP Solar, T1000-E, Nano G2 Ultra, WIO Tracker, Muziworks R1 Neo and the rest — no longer compiles the module in at this release. Those nodes will try to boot on a nearly flat cell and have no low-voltage shutdown or LPCOMP wake, so monitor the battery yourself: see [Reading battery voltage remotely](#reading-battery-voltage-remotely).

!!! note "Historical: v1.17.1 and earlier"
    In v1.17.1 thirteen variants compiled `NRF52_POWER_MANAGEMENT` in:
    RAK4631, RAK3401, Xiao nRF52840, Heltec T114, Heltec T096, Heltec T1,
    Heltec Tower V2, SenseCAP Solar, GAT562 Mesh Watch13, Tracker Pro,
    EVB Pro, 30S Mesh Kit, and Muziworks R1 Neo. Most locked out below
    **3300 mV**; Heltec T1 and Heltec Tower V2 already used 3100 mV, and
    Muziworks R1 Neo set the threshold to 0, which disabled the check.
    Nodes flashed with those releases keep the old behaviour until reflashed.

Internals — LPCOMP reference selection, SYSTEMOFF entry, shutdown reason codes — are covered in the [nRF52 Power Management spec](https://docs.meshcore.io/nrf52_power_management/).

---

## Practical deployment checklist

- [ ] Battery capacity ≥ 3 days of estimated consumption (buffer for cloudy days if solar).
- [ ] Solar panel rated ≥ 2× daily consumption (accounting for panel efficiency and partial shading).
- [ ] `set dutycycle` matches your regional regulatory requirement.
- [ ] `powersaving on` evaluated for your board — check if sleep is implemented.
- [ ] `set flood.advert.interval` set to the longest interval that still meets your network's re-discovery needs.
- [ ] `get pwrmgt.bootmv` checked after first outdoor deployment to confirm the boot voltage is healthy (only on the three boards that still support power management; elsewhere read `stats-core`).
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

As of v1.18.0 `get pwrmgt.bootmv` only answers on the boards listed under [Boards with power management](#boards-with-power-management-at-v1180); elsewhere it returns `ERROR: Power management not supported` and `stats-core` is the only source.

For real-time current voltage, query `stats-core` over serial or from the companion admin CLI stats tab.

If the reported voltage is wrong for your board, calibrate with:

```
get adc.multiplier      — current multiplier (0.0 = use board default)
set adc.multiplier 2.1  — adjust until reading matches a multimeter
```
