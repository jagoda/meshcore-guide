# Remote Admin CLI

Every MeshCore infrastructure node — repeater, room server, sensor — responds to a command-line interface (CLI). You can reach this CLI in two ways:

- **Locally**, via USB serial (picocom, the web flasher console, or config.meshcore.io).
- **Remotely**, over the mesh via the companion app's **Admin** tab.

This page explains both access paths, the command categories, and how to work safely on a live node. For the exhaustive command reference, see the [CLI Commands spec](https://docs.meshcore.io/cli_commands/) and the [Terminal Chat CLI spec](https://docs.meshcore.io/terminal_chat_cli/) on docs.meshcore.io.

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

### Web config UI

[config.meshcore.io](https://config.meshcore.io) provides a form-based UI over USB serial — a friendlier alternative to raw CLI for initial configuration (name, frequency, password, location).

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
| `clock sync` | Sync the node clock from the connecting companion |
| `clock` | Display current UTC time on the node |
| `time <epoch_seconds>` | Set the clock to a specific Unix timestamp |
| `advert` | Send a flood advert now |
| `advert.zerohop` | Send a zero-hop (local-only) advert |
| `start ota` | Initiate over-the-air firmware update |
| `erase` | **Destructive** — factory reset (serial only) |

### Info

| Command | What it does |
|---------|-------------|
| `ver` | Firmware version string |
| `board` | Hardware board name |
| `get role` | Node role (repeater / room_server / sensor) |
| `get public.key` | Node's public key (hex) |
| `get name` | Current node name |
| `get owner.info` | Owner info text |

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
| `get radio.rxgain` / `set radio.rxgain on\|off` | Boosted RX gain (SX12xx, v1.14.1+) | Default: on |
| `tempradio <f>,<bw>,<sf>,<cr>,<mins>` | Temporary radio params | Reverts on reboot |

### Routing configuration

| Command | Default | Notes |
|---------|---------|-------|
| `get/set repeat on\|off` | on | Turn off for observer mode |
| `get/set flood.max <0-64>` | 64 | Cap flood hop count |
| `get/set flood.max.unscoped <n>` | 0xFF (tracks flood.max) | Limit unscoped (no-region) floods |
| `get/set dutycycle <1-100>` | 50 | % duty cycle limit |
| `get/set txdelay <0-2>` | 0.5 | Flood retransmit jitter factor |
| `get/set direct.txdelay <0-2>` | 0.2 | Direct traffic retransmit jitter |
| `get/set loop.detect off\|minimal\|moderate\|strict` | off | Drop looped flood packets (v1.14+) |
| `get/set path.hash.mode 0\|1\|2` | 0 | 1/2/3-byte advert path hash (v1.14+) |
| `get/set flood.advert.interval <hours>` | 12 (repeater) | Flood advert timer |
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
```

### Power management (nRF52, supported boards)

```
get pwrmgt.support      — "supported" or "unsupported"
get pwrmgt.source       — "battery" or "external"
get pwrmgt.bootreason   — reset and shutdown reason strings
get pwrmgt.bootmv       — boot voltage in millivolts
```

See [Power & Battery](power-and-battery.md) and the [nRF52 Power Management spec](https://docs.meshcore.io/nrf52_power_management/) for context.

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
