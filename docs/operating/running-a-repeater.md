# Running a Repeater

A MeshCore repeater is the backbone of any multi-hop network. Unlike Meshtastic, **only dedicated repeater firmware repeats** — companion radios never do. This section walks you from unboxing to a tuned, production-ready repeater.

---

## What a repeater does (and doesn't do)

A repeater listens for every MeshCore packet on the air. For each flood packet it hears it evaluates whether to forward it — checking for duplicates, applying the hop limit, and enforcing duty-cycle rules. For direct (path-routed) packets it inspects the embedded path and forwards only if its own hash appears at the right position.

A repeater does **not**:

- Originate direct messages on behalf of users.
- Store message history (that's a room server's job).
- Repeat every packet unconditionally — duplicate suppression keeps the air clean.

---

## Hardware selection

Any supported LoRa device can run repeater firmware. Prioritise:

- **Antenna port** — an SMA or IPEX connector for an external antenna. PCB antennas are fine for experimentation but limit range in production.
- **Power input** — USB-C, JST LiPo header, or solar-input pins for always-on deployment. See [Power & Battery](power-and-battery.md).
- **Enclosure** — outdoor repeaters need weatherproofing; a project box with a sealed cable gland works well.

For the current supported device list go to [flasher.meshcore.io](https://flasher.meshcore.io).

---

## End-to-end setup guide

### Step 1 — Flash repeater firmware

1. Connect the device to your computer via USB.
2. Open [flasher.meshcore.io](https://flasher.meshcore.io) in Chrome.
3. Choose your board and select **Repeater** firmware.
4. Click **Flash**. The flasher handles offsets automatically.

After flashing the device boots into repeater firmware with default settings.

### Step 2 — First-time serial configuration

Keep the USB cable connected. Open the **Console** tab on the flasher (or use a tool like `picocom`):

```bash
picocom -b 115200 /dev/ttyUSB0 --imap lfcrlf
```

Set the frequency for your region — this is the most important step:

```
set freq 915.525
```

Set a human-readable name:

```
set name MyRepeater
```

Change the default admin password **immediately**:

```
password <your-strong-password>
```

> **Security note:** The default admin password is `password`. Any companion node that knows this password gains admin-level remote CLI access to your repeater. Change it before putting the repeater on the air.

Optionally set location (enables the repeater to appear on [map.meshcore.io](https://map.meshcore.io)):

```
set lat 51.5074
set lon -0.1278
```

Reboot to apply radio settings:

```
reboot
```

### Step 3 — Verify the repeater is on the air

From a companion radio within RF range:

1. Open the companion app → **Contacts**.
2. The repeater should appear after it broadcasts its flood advert (default interval: 47 hours). To see it immediately, trigger an advert from the serial console: `advert`.
3. Tap the repeater entry to verify name and last-seen timestamp look correct.

---

## Remote administration via the companion app

Once the repeater is in your contact list and you know the admin password:

1. Tap the repeater in **Contacts**.
2. Tap **Admin** (or the three-dot menu → **Remote Admin**).
3. Enter the admin password.
4. You now have a remote CLI tab. Type commands as if at a serial console.

> **Freemium note:** Remote admin over RF has a wait-gate on the iOS/Android app — a one-time unlock removes it. The T-Deck requires a registration key for this feature. See [faq.md §3.1](https://docs.meshcore.io/faq/) for details.

!!! note "v1.17: DIRECT admin logins can reply via the stored out_path"
    A companion app's anonymous admin login arrives as a DIRECT packet. As of
    v1.17, if the repeater already has a stored `out_path` for that client
    from an earlier exchange, it replies along that stored path
    (`REPLY_ROUTE_DIRECT_OUT_PATH`) even when the current request didn't
    itself supply a return path — see
    [Routing and Flooding § Reply routing and reply scope](../protocol/routing-and-flooding.md#reply-routing-and-reply-scope-v117).

---

## Key CLI commands for repeater operators

All commands below can be sent locally (serial) or remotely (companion app admin CLI).
For the full command reference see [CLI Commands spec](https://docs.meshcore.io/cli_commands/) on docs.meshcore.io.

!!! note "v1.17: a fourth access path on Ethernet hardware"
    On `RAK_4631_repeater_ethernet` firmware (RAK4631 + RAK13800), the same
    CLI is also reachable over raw TCP on port 23 — see [Remote Admin CLI §
    Ethernet (TCP)](remote-admin-cli.md#ethernet-tcp-v117). `eth.status`
    reports connection state and the DHCP-assigned IP. This is repeater/room
    server only — companion Ethernet firmware exposes no raw CLI at all; its
    Ethernet interface only carries the companion binary protocol (port
    5000), replacing BLE.

### Operational status

```
ver                    — firmware version
board                  — hardware board name
stats-core             — battery voltage, uptime, queue length (serial only)
stats-radio            — noise floor, RSSI/SNR, airtime (serial only)
stats-packets          — packet counters (serial only)
neighbors              — list last 8 heard neighbours with SNR
clock                  — current device time (UTC)
clock sync             — sync clock from the connecting companion
```

### Radio settings

```
get radio              — view current freq / BW / SF / CR
set radio <freq>,<bw>,<sf>,<cr>   — change radio params (reboot to apply)
get tx                 — current transmit power (dBm)
set tx <dbm>           — change TX power (reboot to apply)
```

!!! warning "TX power limits"
    Setting TX power too high can violate regulations and damage hardware with a PA stage. Check your board's manual and your regional EIRP limit before increasing power. The [CLI Commands spec](https://docs.meshcore.io/cli_commands/) lists per-device notes.

### Routing tuning

```
get repeat             — on/off (default: on)
set repeat off         — turn repeating off (observer mode, see below)
get flood.max          — maximum flood hops (default: 64)
set flood.max 32       — cap floods at 32 hops
get flood.max.unscoped — hop cap for unscoped (no-region) floods (default: 64, max 64)
get flood.max.advert   — hop cap for ADVERT floods (default: 8, max 64)
get dutycycle          — duty cycle limit (%)
set dutycycle 10       — limit to 10% (EU 868 MHz requirement)
get txdelay            — retransmit window factor (default: 0.5)
set txdelay 1.0        — widen the window in dense networks
get loop.detect        — loop detection mode (off / minimal / moderate / strict;
                         default: minimal on new installs, upgraded nodes keep their value)
set loop.detect minimal
```

!!! note "v1.18: loop detection now defaults to `minimal`"
    New repeater installs ship with `loop.detect` set to `minimal` instead of
    `off`. In `minimal` mode a flood is dropped once this repeater's own hash
    already appears in the path 4 times (1-byte hashes), 2 times (2-byte) or
    once (3-byte) — the `max_loop_minimal[]` table in
    `examples/simple_repeater/MyMesh.cpp`. Only malformed floods that keep
    re-entering the same node ever reach those counts, which is why it is safe
    as a default. `moderate` and `strict` are unchanged and still opt-in.

    The default lives in the repeater's compiled-in prefs and is then
    overridden by `/prefs.json`, so a repeater **upgraded** from v1.17.1 keeps
    the `off` value already saved in its prefs. After upgrading, run
    `get loop.detect` and set it explicitly if you want the new behaviour;
    only a fresh install (or erasing the filesystem) picks up `minimal`
    automatically. Upgrade steps are in [Daily Use](daily-use.md)
    ("Upgrading to v1.18.0"); if a fresh node seems to forward less flood
    traffic, see [Troubleshooting](troubleshooting.md) ("Repeater drops flood
    packets after a fresh v1.18 install").

As of v1.16 a repeater forwards floods under **separate** caps: regular unscoped
floods are limited by `flood.max.unscoped` (default 64) and ADVERT floods by
`flood.max.advert` (default 8). `flood.max` remains the overall flood-hop cap.
Lowering `flood.max.advert` is the cheapest way to keep advert storms local
without throttling ordinary traffic.

!!! note "v1.17: fixed a reply-drop interaction with a low flood.max.unscoped"
    Before v1.17, a repeater with `flood.max.unscoped` set low (e.g. `0`)
    would silently swallow replies whose requester scope could not be
    resolved — those replies were sent un-scoped and then dropped by this
    node's own low cap, with nothing visible to the client. v1.17 sends that
    reply under the node's default region scope instead, so it is no longer
    subject to `flood.max.unscoped` at all. If you previously worked around
    dropped replies by raising `flood.max.unscoped`, you can now set it back
    down. See [Routing and Flooding § Reply routing and reply
    scope](../protocol/routing-and-flooding.md#reply-routing-and-reply-scope-v117)
    for the full decision table.

### Path hash size (firmware ≥ 1.14)

```
get path.hash.mode     — 0 = 1-byte (default), 1 = 2-byte, 2 = 3-byte
set path.hash.mode 1   — use 2-byte hashes in this repeater's adverts
```

Larger hashes help mesh-analysis tools disambiguate repeaters but reduce max flood range. See [FAQ §3.9](https://docs.meshcore.io/faq/#39-q-what-is-multibyte-support) for migration guidance.

### Flood advert interval

```
get flood.advert.interval      — hours (default: 47)
set flood.advert.interval 6    — advert every 6 hours
get advert.interval            — minutes between zero-hop adverts
```

!!! note "v1.18: a repeater with adverts turned off is now genuinely hidden"
    Turning **both** timers off — `set advert.interval 0` (zero-hop) and
    `set flood.advert.interval 0` — makes the repeater a *hidden* node (in code,
    `MyMesh::isHiddenNode()` returns true only when `advert_interval` and
    `flood_advert_interval` are both 0). Advert *sending* is unchanged by this
    release; what changed is that v1.18 also adds an `!isHiddenNode()` check to
    node-discovery request handling, so `onControlDataRecv()` now bails out
    early for hidden nodes. In v1.17.1 the repeater kept answering
    `CTL_TYPE_NODE_DISCOVER_REQ`, so it stayed findable by anyone in direct
    radio range running a discovery scan in the companion app.

    Hiding is not the same as going silent or passive: a hidden repeater still
    relays traffic. That is `set repeat off` (see [Observer
    mode](#observer-mode)). Note also that zero-hop adverts are switched off
    automatically the first time you save any setting, because the CLI resets
    an interval below the 60-minute minimum to 0 (`MIN_LOCAL_ADVERT_INTERVAL`
    in `src/helpers/CommonCLI.cpp`) — that behaviour predates v1.18. See
    [Adverts Deep Dive § Hidden repeaters
    (v1.18)](../protocol/adverts-deep-dive.md#hidden-repeaters-v118) for the
    discovery mechanism and [Troubleshooting](troubleshooting.md) ("A hidden
    repeater no longer appears in node discovery") for the symptom-and-fix
    view.

---

## Observer mode

An **observer** participates in the mesh passively — it hears and records packets but does not forward them. Useful for mesh analysis and coverage mapping.

To set a repeater to observer mode:

```
set repeat off
reboot
```

!!! note
    Observer onboarding instructions for the LetsMesh.net Analyzer are at [analyzer.letsmesh.net/observer/onboard](https://analyzer.letsmesh.net/observer/onboard).

---

## Troubleshooting a new repeater

| Symptom | Likely cause | Remedy |
|---------|-------------|--------|
| Repeater not seen in contacts | Wrong frequency | `get radio`; set freq to match other nodes |
| Repeater seen but messages don't relay | `set repeat` is off | `get repeat`; `set repeat on` |
| Old last-seen timestamp | Clock not set | `clock sync` from a companion admin session |
| Repeater never appears in a node-discovery scan | Node is hidden — both advert timers at 0 (v1.18 behaviour) | `get advert.interval` and `get flood.advert.interval`; set one non-zero. See [Troubleshooting](troubleshooting.md) — *A hidden repeater no longer appears in node discovery* |
| Deafness — not hearing nearby nodes | AGC stuck | `set agc.reset.interval 4` |
| Lots of duplicate forwards | txdelay too low for dense mesh | Increase `txdelay` |

For more diagnosis flows see [Troubleshooting](troubleshooting.md).
