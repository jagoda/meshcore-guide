# Troubleshooting

This page covers the most common problems operators encounter and the steps to diagnose them. For additional Q&A see the [MeshCore FAQ](https://docs.meshcore.io/faq/) on docs.meshcore.io.

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

## nRF52 device won't boot / stuck in boot loop

**Low battery:** The nRF52 power management firmware prevents boot below ~3300 mV to avoid flash corruption. The device enters SYSTEMOFF and wakes when the battery recovers or USB is connected.

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
