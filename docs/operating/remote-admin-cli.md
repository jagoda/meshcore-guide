# Remote Admin CLI

Every MeshCore infrastructure node — repeater, room server, sensor — responds to a command-line interface (CLI). You can reach this CLI in three ways:

- **Locally**, via USB serial (picocom, the web flasher console, or config.meshcore.io).
- **Remotely**, over the mesh via the companion app's **Admin** tab.
- **Over Ethernet** (v1.17, RAK4631 + RAK13800 hardware only) — a wired TCP alternative to the two above.

This page explains all the access paths, the command categories, and how to work safely on a live node. For the exhaustive command reference, see the [CLI Commands spec](https://docs.meshcore.io/cli_commands/) and the [Terminal Chat CLI spec](https://docs.meshcore.io/terminal_chat_cli/) on docs.meshcore.io.

!!! note "Companion firmware also has a CLI as of v1.18"
    v1.18.0 adds a text CLI to **companion** firmware, reached over the
    companion's existing BLE/USB/Wi-Fi link or remotely over the mesh from a
    contact granted the new **Remote CLI** permission. It is a separate
    command path from the infrastructure CLI documented below — see
    [Companion CLI](#companion-cli-v118).

---

## Access paths

### USB serial (local)

Connect the node via USB. Use any serial terminal at **115200 baud**:

```bash
# Linux / macOS
picocom -b 115200 /dev/ttyUSB0 --imap lfcrlf

# Raspberry Pi / nRF devices often use ACM
picocom -b 115200 /dev/ttyACM0 --imap lfcrlf
```

Or use the browser-based console at [config.meshcore.io](https://config.meshcore.io) (Chrome, WebSerial required). This is the only way to access **serial-only commands** (e.g. `get acl`, `stats-core`, `erase`, `get prv.key`).

### Remote admin via companion app

1. The target node must be in your **Contacts** list (i.e. you've heard its advert).
2. Tap the node → three-dot menu → **Remote Admin**.
3. Enter the node's **admin password** (default: `password`; change this before deployment).
4. Type commands in the admin CLI tab. Responses arrive over the mesh.

!!! note "Freemium gate"
    Remote admin over RF has a wait-timer gate in the iOS/Android companion app. A one-time in-app purchase removes the wait. The T-Deck requires a registration key. Neither is needed for the core messaging experience.

### Ethernet (TCP) (v1.17)

On **RAK4631 + RAK13800** hardware flashed with an `_ethernet` firmware
variant, DHCP assigns an IP address on boot. Repeater and room server
firmware expose the same CLI over raw **TCP port 23** — connect with
`nc <ip> 23` or a raw-mode terminal, no companion app involved. Check
`eth.status` (repeater/room only) for connection state and assigned IP.

Companion Ethernet firmware (`RAK_4631_companion_radio_ethernet`) works
differently: Ethernet **replaces BLE** as the companion app's transport, on
**TCP port 5000** (the same port the Wi-Fi companion firmware uses) — there
is no raw CLI over Ethernet on companion firmware, and no BLE fallback once
you've flashed the Ethernet variant.

See [What You Need § Wired backhaul](../getting-started/what-you-need.md) and
[Connect a Client](../getting-started/connect-a-client.md) for setup, and
[Troubleshooting](troubleshooting.md) for no-DHCP / can't-reach-port-23
symptoms.

### Web config UI

[config.meshcore.io](https://config.meshcore.io) provides a form-based UI over USB serial — a friendlier alternative to raw CLI for initial configuration (name, frequency, password, location).

### Companion CLI (v1.18)

Companion firmware gained its own CLI in v1.18.0. Commands travel over an
established companion connection rather than a TCP port: locally through the
companion app's command-line screen or a serial/USB app session, or remotely
over the mesh from a contact whose **Remote CLI** permission is enabled. The
protocol additions behind it (`CMD_RUN_CLI_COMMAND`, `RESP_CODE_CLI_REPLY` and
companion protocol version code 14) are documented in [Command
Lifecycle](../companion-api/command-lifecycle.md) rather than repeated here.

The command set reuses the infrastructure radio-preference commands (`radio`,
`af`, `dutycycle`, `int.thresh`, `cad`, `radio.rxgain`, `tx`, `rxdelay`,
`txdelay`, `direct.txdelay`, `agc.reset.interval`, `path.hash.mode`,
`multi.acks`, and the board-gated FEM gains), plus these companion-specific
commands added in v1.18:

| Command | Notes |
|---------|-------|
| `get/set tz.offset <-12..+14>` (v1.18) | Whole-hour offset used by the companion's clock display. Stored in preferences; default `0`. |
| `get/set wifi.ssid <ssid>` / `get/set wifi.pwd <password>` (v1.18) | Wi-Fi-capable builds only. Reboot to apply. `get wifi.pwd` returns the stored password in clear text. |
| `get wifi.status` / `get wifi.ip` (v1.18) | `connected` or `disconnected`, and the current address when connected. |
| `get/set wifi.enabled 0\|1` (v1.18) | Reboot to apply. Enabled by default, but Wi-Fi only starts when an SSID is resolved (stored or compile-time). |
| `set wifi.clear` (v1.18) | Erase the stored SSID and password; reboot to apply. |
| `reboot`, `poweroff`, `shutdown` (v1.18) | New on companion firmware in v1.18. They do not send a reply. |

---

## Authentication and the admin password

Any companion node that supplies the correct admin password gains admin-level CLI access over the mesh. This is powerful and intentional — it's how you administer a remote repeater on a hilltop without hiking up to it.

**Best practice:**

1. Change the default password immediately after flashing: `password <your-strong-password>`.
2. Use a strong, unique password per node.
3. The admin password is echoed back in the CLI reply as confirmation.

The password is stored in node preferences (survives reboot; lost on `erase`).

**Guest access:** Room servers have a separate **guest password** (`set guest.password`) for joining the room. Admin and guest passwords are independent.

---

## Command categories

### Operational

| Command | What it does |
|---------|-------------|
| `reboot` | Restart the node |
| `clkreboot` | Reset the clock then reboot |
| `poweroff` / `shutdown` (v1.17) | Power the node down |
| `clock sync` | Sync the node clock from the connecting companion |
| `clock` | Display current UTC time on the node |
| `time <epoch_seconds>` | Set the clock to a specific Unix timestamp |
| `advert` | Send a flood advert now |
| `advert.zerohop` | Send a zero-hop (local-only) advert |
| `start ota` | Initiate over-the-air firmware update |
| `erase` | **Destructive** — factory reset (serial only) |

!!! note "No reply is sent for reboot, clkreboot, poweroff, or shutdown"
    All four commands make the firmware restart or power off before it can
    send a CLI response — the command simply doesn't return. Don't wait on a
    reply from any of them; confirm success by polling `ver` after a short
    delay instead.

### Info

| Command | What it does |
|---------|-------------|
| `ver` | Firmware version string |
| `board` | Hardware board name |
| `get role` | Node role (repeater / room_server / sensor) |
| `get public.key` | Node's public key (hex) |
| `get name` / `set name <name>` | Node name advertised on the mesh | Budget: 23 bytes with location, 31 without (32-byte `app_data` minus flags minus lat/lon); truncated at a UTF-8 code-point boundary, never mid-character (v1.17) |
| `get owner.info` | Owner info text |
| `power` (v1.18) | Battery millivolts plus USB, solar-charger, external-power and charger state; inputs the board cannot sense report `n/a` |

### Statistics (serial only)

| Command | What it shows |
|---------|--------------|
| `stats-core` | Battery voltage, uptime, TX queue length |
| `stats-radio` | Noise floor, last RSSI/SNR, total airtime, RX errors |
| `stats-packets` | Flood/direct packet send and receive counts |
| `clear stats` | Reset all counters |

### Logging

| Command | What it does |
|---------|-------------|
| `log start` | Begin capturing the RX log to flash |
| `log stop` | End capture |
| `log` | Print captured log to serial terminal (serial only) |
| `log erase` | Delete the stored log |

### Radio configuration

| Command | What it does | Notes |
|---------|-------------|-------|
| `get radio` | Current freq, BW, SF, CR | |
| `set radio <freq>,<bw>,<sf>,<cr>` | Change all radio params | Reboot to apply |
| `get tx` / `set tx <dbm>` | TX power in dBm | Reboot to apply; check board limits |
| `get freq` / `set freq <MHz>` | Frequency only | Serial-only for set |
| `get radio.rxgain` / `set radio.rxgain on\|off` | Boosted RX gain (SX12xx and LR1110, v1.14.1+) | Default: on. No longer compile-gated (v1.17) — replies `Error: unsupported` on radios that lack it, rather than being absent from the CLI entirely |
| `get/set cad on\|off` (v1.17) | Hardware Channel Activity Detection scan before TX; defers if the channel is busy | Default: **off**. Runs independently of `int.thresh` — either, both, or neither may be active. Companion firmware was hard-wired off before v1.18; it now exposes the same `get/set cad` through its [Companion CLI](#companion-cli-v118) (v1.18) |
| `get/set radio.fem.rxgain on\|off` (v1.17) | External LoRa FEM receive-path LNA, board-gated | Separate from `radio.rxgain` (radio-chip gain, not the external FEM). `Error: unsupported` on boards without an FEM |
| `get/set radio.fem.txgain on\|off` (v1.17) | External LoRa FEM transmit-path gain, board-gated | On Station G3: remove the PA PL1 jumper for software control; `on` = PL1 high/short, `off` = PL1 low/open — the PL2 hardware jumper picks which power-level pair this switches between. The level is applied at the **start of the next TX**, not immediately, so `get` reports the *configured* state, which may lead the hardware until the node next transmits |
| `get/set extra.sf <sf>[,<sf>[,<sf>]]` (v1.17) | Up to 3 side-detector spreading factors for simultaneous extra-SF reception | **LR2021 radio only** — compiled out (and CLI-absent) on other radio families |
| `tempradio <f>,<bw>,<sf>,<cr>,<mins>` | Temporary radio params | Reverts on reboot |

`set af` is range-checked as of v1.18: values outside `0`–`9` are rejected with
`ERROR: af must be 0-9`. `set agc.reset.interval` is also clamped to 0–1020
seconds before rounding down to a multiple of 4 (v1.18). Both commands are
available on companion firmware through the Companion CLI as well.

### Routing configuration

| Command | Default | Notes |
|---------|---------|-------|
| `get/set repeat on\|off` | on | Turn off for observer mode |
| `get/set flood.max <0-64>` | 64 | Cap flood hop count |
| `get/set flood.max.unscoped <0-64>` | 64 | Cap hop count for unscoped (no-region) floods (separate from `flood.max`) |
| `get/set flood.max.advert <0-64>` | 8 | Cap hop count for ADVERT floods (separate from `flood.max`) |
| `get/set dutycycle <1-100>` | 50 | % duty cycle limit |
| `get/set rxdelay <0-20>` | — | Receive-window delay base |
| `get/set txdelay <0-2>` | 0.5 | Flood retransmit jitter factor |
| `get/set direct.txdelay <0-2>` | 0.3 (repeater, new installs); 0.2 (room server, sensor) | Direct traffic retransmit jitter (repeater default changed in v1.17) |
| `get/set loop.detect off\|minimal\|moderate\|strict` | minimal (repeater, new installs, v1.18); existing nodes keep their value | Drop looped flood packets (v1.14+). `off` disables the check |
| `get/set path.hash.mode 0\|1\|2` | 0 | 1/2/3-byte advert path hash (v1.14+) |
| `get/set flood.advert.interval <hours>` | 47 (repeater) | Flood advert timer |
| `get/set advert.interval <minutes>` | 0 | Zero-hop advert timer |
| `get/set agc.reset.interval <secs>` | 0 | Periodic AGC reset (deafness cure) |
| `get/set int.thresh <value>` | 0 | Local interference threshold |

### ACL (Access Control List)

| Command | What it does |
|---------|-------------|
| `get acl` | View full ACL (serial only) |
| `setperm <pubkey> <0\|1\|2\|3>` | Set a client's permission level (0=Guest, 1=RO, 2=RW, 3=Admin) |
| `setperm <pubkey>` | Remove client from ACL |
| `get allow.read.only` | Room server: whether RO clients may join |
| `set allow.read.only on\|off` | Room server: toggle RO access |

### Region management (v1.10+)

Regions allow repeaters to scope flood traffic geographically. A detailed walk-through is in the [CLI Commands spec](https://docs.meshcore.io/cli_commands/#region-management-v110).

Quick patterns:

```
# Define a region hierarchy
region def Europe UK London|Europe France Paris
region save
```

`region def` takes a list of space-separated **segments**. Each segment is a
region name, optionally followed by a *jump* target: `name|jump` or `name,jump`
(v1.16 accepts both `|` and `,` as the separator). Without a jump, the next
segment nests under the one just defined; a jump repositions the cursor to an
already-defined region so you can start a new branch. In the example above,
`London|Europe` defines `London` under `UK`, then jumps the cursor back to
`Europe` so `France` and `Paris` form a second branch.

```

# Allow a region's packets to flood
region allowf Europe

# Block a region
region denyf NoisyNeighbour

# Set this node's home region
region home Europe

# View all defined regions
region
```

### Sensor commands (sensor firmware only)

```
sensor list             — list all sensor channels and values
sensor get <key>        — read a single value
sensor set <key> <val>  — write a sensor setting
gps                     — GPS status
gps on / gps off        — enable/disable GPS
gps sync                — sync clock from GPS
gps setloc              — store current GPS fix as static location
gps advert <policy>     — none | share | prefs
gps diag                — req/en flags, satellites and fix state; NMEA providers add UART/NMEA counters and ages (v1.18)
```

### Power management (nRF52, supported boards)

```
get pwrmgt.support      — "supported" or "unsupported"
get pwrmgt.source       — "battery" or "external"
get pwrmgt.bootreason   — reset and shutdown reason strings
get pwrmgt.bootmv       — boot voltage in millivolts
```

The commands differ in how they behave when a board has no power management
compiled in (v1.18.0):

| Command | On a board without `NRF52_POWER_MANAGEMENT` |
|---------|---------------------------------------------|
| `get pwrmgt.support` | `> unsupported` |
| `get pwrmgt.source` | `ERROR: Power management not supported` |
| `get pwrmgt.bootmv` | `ERROR: Power management not supported` |
| `get pwrmgt.bootreason` | Not ifdef-gated: replies `> Reset: Not available; Shutdown: Not available` on boards that do not override the reason hooks (ESP32 boards supply a real reset reason, but still `Shutdown: Not available`) |

As of v1.18.0, power management is compiled in only for Heltec T1, Heltec
Tower V2 and Muzi Base; those three use a 3100 mV boot lockout. Every other
nRF52 board boots regardless of battery voltage. See [Power & Battery § nRF52
power management](power-and-battery.md#nrf52-power-management-hardware-level)
for the per-board table and the v1.17.1 history.

Internals — LPCOMP reference selection, SYSTEMOFF entry and shutdown reason codes — are covered in the [nRF52 Power Management spec](https://docs.meshcore.io/nrf52_power_management/).

---

## Working safely on a live node

- **Test radio changes locally first.** A wrong frequency will cut you off from remote access; fix it with serial.
- **Don't `erase` remotely.** It's serial-only for good reason — it would factory-reset a node you can't physically reach.
- **Write down the admin password.** There's no password recovery over RF; serial + `erase` is the reset path.
- **Tempradio for experiments.** Use `tempradio` to trial new radio parameters; they revert automatically on reboot so a bad setting doesn't strand the node.
- **`reboot` to apply radio/system changes.** Many `set` commands are written to preferences but only activated on next boot.

---

## Neighbours (repeater only)

```
neighbors                      — list last 8 heard neighbours
neighbor.remove <pubkey-prefix> — remove a specific neighbour
discover.neighbors             — trigger zero-hop neighbour discovery
```

Output format per line: `{pubkey-prefix}:{timestamp}:{snr×4}`. Divide the SNR field by 4 to get dB.

---

## Quick workflow: commissioning a new repeater remotely

```
# Confirm it's alive
ver
board

# Set name (if not already set)
set name MyHillRepeater

# Check radio params
get radio

# Change admin password
password <strong-password>

# Sync clock
clock sync

# Confirm time
clock

# Check stats
stats-core

# Trigger advert so other nodes see the new config
advert
```
