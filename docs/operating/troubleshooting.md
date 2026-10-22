# Troubleshooting

This page covers the most common problems operators encounter and the steps to diagnose them. For additional Q&A see the [MeshCore FAQ](https://docs.meshcore.io/faq/) on docs.meshcore.io.

**Applies to:** MeshCore firmware v1.18.0; sections note where behaviour differs in earlier releases.

---

## A node shows a very old "last seen" timestamp

**Root cause: clock skew.**

MeshCore timestamps are absolute Unix epoch values. If your node's clock is wrong — or the remote node's clock is wrong — last-seen times appear absurd.

**Diagnosis:**

```
clock           — display the node's current UTC time
```

Compare with a known-good source (phone, NTP).

**Fix:**

```
clock sync      — companion syncs node clock from phone time (remote or local admin CLI)
time <epoch>    — set clock manually; get epoch from epochconverter.com
```

For GPS-equipped nodes: `gps sync` sets the clock from GPS time.

---

## A node I expect to see isn't appearing in my contact list

**Possible causes:**

1. **Wrong frequency.** The other node is on a different frequency. `get radio` on both nodes; ensure they match exactly.
2. **No advert received.** The node hasn't broadcast an advert yet, or its flood advert interval is very long.
   - Trigger a manual advert from the serial console: `advert`
   - Reduce the flood advert interval: `set flood.advert.interval 6` (hours)
3. **Out of RF range.** No path exists between you and the node, even via repeaters.
4. **Clock skew so severe that the advert is filtered.** Sync clocks on both ends.

---

## A hidden repeater no longer appears in node discovery (v1.18)

**Symptom:** after upgrading, a repeater you deliberately made "hidden" no
longer appears when a companion scans for nodes, or in another repeater's
`discover.neighbors` output, even though it still forwards traffic.

**Cause:** a repeater is hidden when both `advert.interval` and
`flood.advert.interval` are `0`. Since v1.18 it also stops answering zero-hop
node-discovery requests, so hiding its adverts now hides it from discovery too.
This is intentional.

**Fix:** if it should be discoverable, give it an advert timer — for example
`set flood.advert.interval 47` (hours) or `set advert.interval 60` (minutes) —
then `reboot`. If it should stay hidden, add it as a contact manually or reach
it by its known route; a manual `advert` from its CLI still works.

---

## My repeater is "deaf" — it's not hearing nearby nodes

**Root cause: SX1262/SX1268 AGC (Automatic Gain Control) can lock up** after being saturated by a strong out-of-band signal. The radio appears alive but doesn't decode packets.

**Diagnosis:** Nearby nodes hear each other fine but this repeater doesn't appear in their `neighbors` output.

**Fix:**

```
set agc.reset.interval 4    — reset AGC every 4 seconds (very low cost, highly effective)
```

This periodically re-initialises the radio's gain control. Set it once and save; the node will no longer suffer from prolonged deafness episodes.

---

## Messages aren't getting through / lots of retries

**Step 1: Check the path.** The companion app retries twice on a stored path before flooding. If the stored path is stale, clear it:

- App: long-press contact → **Reset Path**
- CLI: `reset path`

**Step 2: Check RF conditions.** Open `stats-radio` (serial) to see:

- **Noise floor** — if it's high (-90 dBm or above), there's local interference.
- **Last RSSI/SNR** — weak signal means marginal path.

**Step 3: Check duty cycle.** If the duty cycle limit is hit, the node holds transmissions:

```
get dutycycle       — view current limit
stats-radio         — check total airtime
```

If airtime is saturated, either reduce `dutycycle` (if regulation allows) or reduce traffic sources.

**Step 4: Check for a packet storm / loop.** If `stats-packets` shows an unusual flood of received packets, a rogue node may be causing a loop. Enable loop detection:

```
set loop.detect minimal     — drop packets that appear in a loop
```

---

## Repeater drops flood packets after a fresh v1.18 install

**Symptom:** a newly flashed or factory-reset v1.18 repeater forwards less
flood traffic than expected; `stats-packets` shows received floods that are not
being forwarded.

**Cause:** new repeater installs now default `loop.detect` to `minimal`
(v1.18). It drops a flood packet when the repeater's own hash already appears
in the path 4 times with 1-byte hashes, 2 times with 2-byte hashes, or once
with 3-byte hashes. Upgraded repeaters keep their previous setting, so this
affects fresh installs and factory resets only.

**Fix:** confirm with `get loop.detect`. If your mesh depends on the old
forward-everything behaviour, `set loop.detect off`; otherwise leave it on as
protection against loop storms.

---

## Path-hash collisions — messages reach wrong node or stats show unexpected paths

**Root cause:** Two or more repeaters share the same 1-byte public-key prefix (first byte of their public key). Path analysis tools and traceroutes become ambiguous.

**This does not break messaging** — packets still reach their destinations, but path analysis is harder.

**Solutions:**

1. **Generate a unique key:** Use [gessaman.com/mc-keygen](https://gessaman.com/mc-keygen/) to create a private key whose public key starts with an unused byte. Set it via serial: `set prv.key <hex>` then `reboot`.

2. **Upgrade to multibyte path hashes (firmware ≥ 1.14):**
   ```
   set path.hash.mode 1    — 2-byte advert hash (65,536 unique IDs)
   set path.hash.mode 2    — 3-byte advert hash (16.7M unique IDs)
   ```
   Wait until the majority of repeaters in your region are on firmware ≥ 1.14 before switching channel/direct messages to multibyte; older firmware drops those packets silently. See [FAQ §3.9](https://docs.meshcore.io/faq/#39-q-what-is-multibyte-support) for migration guidance.

---

## GPS lock issues (T-Deck, GPS-equipped nodes)

**T-Deck Plus not getting a fix:**

- Set GPS baud rate to **38400** (T-Deck Plus uses a different module than OG T-Deck).
- If still no fix, check physical GPS module orientation — some T-Deck Plus units shipped with the module installed upside down (antenna facing down). Open the case to verify.

**OG (non-Plus) T-Deck:**

- OG T-Deck has no built-in GPS. If you added one externally, try baud rates from 9600 up to 115200 until the `Sentences:` counter on the GPS Info screen increments.

**`gps` CLI (sensor/repeater with GPS support compiled in):**

```
gps                 — shows: off | on, {active|deactivated}, {fix|no fix}, {sat count} sats
gps on              — enable GPS hardware
gps sync            — sync clock from GPS time once a fix is acquired
gps setloc          — write current GPS coordinates to the node's stored lat/lon
gps diag            — UART/NMEA counters, ages, satellite count and fix state (v1.18)
```

---

## Bluetooth connection issues (companion radio)

| Problem | Cause | Fix |
|---------|-------|-----|
| Companion not visible in BLE scan | USB firmware flashed instead of BLE | Re-flash with **BLE Companion** firmware |
| Pairing fails | Wrong code | Default BT pairing code is `123456` |
| Heltec V3 drops BLE frequently | Tiny PCB coil antenna | Replace coil with 31 mm wire antenna for better range |
| nRF device not connecting | Out of OTA/DFU mode or corrupted flash | Double-press reset to enter DFU mode; re-flash |

---

## Can't connect to repeater over Bluetooth

**This is by design.** Devices running **repeater firmware** do not expose a BLE interface. Only **companion radio firmware** (BLE variant) connects via Bluetooth.

To administer a repeater remotely, use the companion app's **Remote Admin** tab over the mesh (requires the admin password and the repeater to be in your contact list).

---

## Ethernet node not reachable (no DHCP address / can't connect to port 23)

**Applies to:** `_ethernet` firmware variants on RAK4631 + RAK13800 (v1.17, unchanged in v1.18).

- **No DHCP server on the network.** The Ethernet firmware does not fall back
  to a static or link-local address — if DHCP doesn't respond, the device
  never gets an IP and Ethernet access does not come up at all. Check `eth.status`
  over USB serial to confirm whether an address was assigned; if not, verify
  a DHCP server is reachable on that network segment.
- **Port 23 (repeater/room server) unreachable but the device has an IP.**
  Confirm you're using a raw TCP client (`nc <ip> 23`), not an SSH or Telnet
  client that expects negotiation — the CLI is a bare byte stream, not the
  Telnet protocol. Check for a firewall blocking port 23 between your machine
  and the device.
- **Companion app can't connect over Ethernet.** Companion Ethernet firmware
  listens on **port 5000**, not 23 — port 23 CLI access does not exist on
  companion firmware at all (see [Remote Admin CLI §
  Ethernet](remote-admin-cli.md#ethernet-tcp-v117)).
- **BLE stopped working after flashing an Ethernet companion variant.** This
  is expected — Ethernet **replaces** BLE on `RAK_4631_companion_radio_ethernet`;
  there is no BLE fallback on that firmware.

---

## Settings reverted after downgrading from v1.17 to v1.16

This is the legacy-prefs-file asymmetry, not a bug: v1.17 stores preferences
in a new `/prefs.json` file but deliberately leaves the old binary prefs file
in place when it migrates. A downgrade back to v1.16 reads that old file
again, so any settings changed *while running v1.17* — a new admin password,
a renamed node, a radio retune — silently revert to whatever they were at the
moment of the v1.16→v1.17 upgrade. Re-apply the changes you made under v1.17
after downgrading, or avoid downgrading a node whose settings have since
changed. See [Daily Use § Upgrade history: v1.16 to
v1.17](daily-use.md#upgrade-history-v116-to-v117).

---

## Settings added in v1.18 are ignored after downgrading to v1.17

v1.18 and v1.17 both use `/prefs.json`, so a v1.18→v1.17 downgrade does not
revert the whole file. v1.17 simply ignores keys it does not know — the v1.18
Wi-Fi credentials and `tz.offset`, for example — so those settings appear to be
lost. Saving any setting under v1.17 rewrites `/prefs.json` without the unknown
keys, so copy them down before downgrading if you intend to come back to v1.18.

---

## nRF52 device won't boot / stuck in boot loop

**Low battery:** On the nRF52 builds that still enable power management (Heltec T1, Heltec Tower V2 and Muzi Base as of v1.18), firmware prevents boot below ~3300 mV to avoid flash corruption. The device enters SYSTEMOFF and wakes when the battery recovers or USB is connected. Most other nRF52 boards had this boot lockout disabled in v1.18, so a low battery can let the device start and then brown out mid-operation; charge it and check `get pwrmgt.bootmv` before relying on it.

**Corrupted flash:** Enter DFU mode by double-pressing the reset button (RAK, T114, XIAO) or double-tapping the magnetic connector (T1000-E). Then:

1. Download `flash_erase*.uf2` for your board from [flasher.meshcore.io](https://flasher.meshcore.io).
2. Copy it to the DFU drive that appears on your computer.
3. Re-flash the MeshCore firmware.

Check boot reason after recovery:

```
get pwrmgt.bootreason       — e.g. "Boot protect / Boot Voltage"
get pwrmgt.bootmv           — mV at last boot
```

---

## WebFlasher fails on Linux ("failed to open serial port")

The browser user doesn't have permission for the serial device:

```bash
sudo setfacl -m u:$USER:rw /dev/ttyUSB0
# or for nRF devices:
sudo setfacl -m u:$USER:rw /dev/ttyACM0
```

---

## Interference and noise floor

High noise floor (-90 dBm or above on `stats-radio`) reduces effective range and increases packet errors.

**Diagnoses:**

- `stats-radio` → note `noise_floor` and `n_recv_errors`.
- `get radio` → confirm you're on the correct regional preset. Many regions moved to BW62.5 / SF7–9 (narrower bandwidth fits between interference sources better than the original BW250/SF11).

**Mitigations:**

- Move to a narrower bandwidth preset if your region supports it.
- Change frequency slightly (`set freq`, reboot) to avoid a specific interferer.
- Increase coding rate (`set radio <freq>,<bw>,<sf>,7`) for more error correction at the cost of throughput.
- Set `set agc.reset.interval 4` to prevent AGC lockup from strong interferers.
- Reduce `set int.thresh` to have the repeater filter locally-generated interference before forwarding.

---

## Useful diagnostic command sequence

Run this from a serial console to get a full health picture of an infrastructure node:

```
ver                 — firmware version
board               — hardware board name
get role            — repeater / room_server / sensor
clock               — current UTC time
get radio           — radio parameters
get tx              — TX power
get dutycycle       — duty cycle limit
stats-core          — battery, uptime, queue (serial only)
stats-radio         — noise floor, RSSI/SNR, airtime (serial only)
stats-packets       — send/recv counters (serial only)
neighbors           — last 8 heard neighbours with SNR (repeater only)
get pwrmgt.source   — battery or external (nRF52 boards with pwr mgmt)
get pwrmgt.bootmv   — boot voltage (nRF52 boards with pwr mgmt)
power               — battery, USB, solar-charger and external-power state (v1.18)
```

---

## Where next in your journey

You have completed the Operating MeshCore section. Depending on your goal:

- **Stay as an operator** — revisit [Running a Repeater](running-a-repeater.md)
  or [Remote Admin CLI](remote-admin-cli.md) as reference when problems arise.
- **Go deeper into how MeshCore works** — [The Protocol →](../protocol/index.md)
  explains the packet anatomy, routing logic, and encryption model that underlies
  everything you just configured. This is the gateway to the developer track.
- **Build on MeshCore** — after The Protocol, continue to
  [Companion API](../companion-api/index.md) and
  [Architecture & Internals](../internals/index.md).
