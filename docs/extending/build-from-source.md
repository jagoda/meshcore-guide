# Build from Source

Pre-built firmware covers most deployments. Build from source when you need a
custom variant, a debug build, a non-standard configuration, or when you're
developing a feature for contribution.

## Prerequisites

| Tool | Version | Notes |
|---|---|---|
| Python 3 | ≥ 3.9 | Required by PlatformIO CLI and `build.sh` helpers |
| [PlatformIO Core](https://docs.platformio.org/en/latest/core/installation/) | ≥ 6 | `pip install platformio` or via IDE plugin |
| Git | any | For fetching the repo and resolving the commit hash embedded in the firmware version string |
| Bash | any | macOS/Linux natively; Windows users: Git Bash or WSL |

## Clone the repo

```bash
git clone https://github.com/meshcore-dev/MeshCore.git
cd MeshCore
```

The `dev` branch is the active development branch; `main` tracks the most recent
release. For building production firmware, prefer a tagged release.

## Discover available targets

`build.sh` queries PlatformIO for every `[env:...]` section across the root
`platformio.ini` and all `variants/*/platformio.ini` files:

```bash
sh build.sh list
```

Typical output (excerpt):

```
Heltec_v3_repeater
Heltec_v3_companion_radio_usb
Heltec_v3_companion_radio_ble
Heltec_v3_sensor
Heltec_v3_kiss_modem
RAK_4631_repeater
RAK_4631_companion_radio_usb
...
```

Target names follow the pattern `<board>_<role>`. Roles are:

| Suffix | Role |
|---|---|
| `_repeater` | Relay node — forwards packets, no companion API |
| `_companion_radio_usb` / `_companion_radio_ble` | Companion radio — runs the serial/BLE companion API |
| `_room_server` | Room server — persistent channel BBS |
| `_sensor` | Sensor node — telemetry and alerting |
| `_kiss_modem` | KISS TNC — raw packet interface for external software |

## Set the firmware version

`build.sh` requires a `FIRMWARE_VERSION` environment variable. It is embedded in
the binary and exposed in the node's status response.

```bash
export FIRMWARE_VERSION=v1.16.0   # or any string you like
```

## Build a single target

```bash
sh build.sh build-firmware Heltec_v3_repeater
```

Output files land in `out/`:

| Platform | Files produced |
|---|---|
| ESP32 | `<target>-<version>.bin`, `<target>-<version>-merged.bin` |
| nRF52 | `<target>-<version>.uf2`, `<target>-<version>.zip` |
| STM32 | `<target>-<version>.bin`, `<target>-<version>.hex` |
| RP2040 | `<target>-<version>.bin`, `<target>-<version>.uf2` |

The `-merged.bin` (ESP32 only) bundles the bootloader, partition table, and
application into a single file for use with `esptool.py` or the web flasher on a
fresh device.

## Build multiple targets

```bash
# All firmwares (companion + repeater + room server)
sh build.sh build-firmwares

# Only repeaters
sh build.sh build-repeater-firmwares

# Only companion radios
sh build.sh build-companion-firmwares

# Targets whose name contains a substring
sh build.sh build-matching-firmwares RAK_4631
```

## Disable debug logging

Debug logging (serial output, packet traces) is controlled by preprocessor flags
set inside each variant's `platformio.ini`. Strip them all at build time with:

```bash
export DISABLE_DEBUG=1
sh build.sh build-firmware RAK_4631_repeater
```

When `DISABLE_DEBUG=1`, `build.sh` appends `-U` undefines for `MESH_DEBUG`,
`MESH_PACKET_LOGGING`, `BLE_DEBUG_LOGGING`, `WIFI_DEBUG_LOGGING`, and several
others — leaving the binary lean for production.

## How the build system is wired

```
platformio.ini (root)
  extra_configs = variants/*/platformio.ini   ← auto-includes every variant
  [arduino_base]   ← shared flags (RadioLib, Crypto, CayenneLPP, etc.)
  [esp32_base]     ← extends arduino_base
  [nrf52_base]     ← extends arduino_base
  [rp2040_base]    ← extends arduino_base

variants/heltec_v3/platformio.ini
  [Heltec_lora32_v3]      ← board-level config (SPI pins, radio class, TX power)
    extends = esp32_base
  [env:Heltec_v3_repeater]
    extends = Heltec_lora32_v3
    build_src_filter += <../examples/simple_repeater>
  [env:Heltec_v3_sensor]
    extends = Heltec_lora32_v3
    build_src_filter += <../examples/simple_sensor>
  ...
```

`build_src_filter` entries tell PlatformIO which source trees to compile.
`build_flags` carry `#define` constants (pin numbers, radio class, feature flags)
into the C++ preprocessor. Both cascade via `extends`.

## Direct PlatformIO invocation

You can also drive PlatformIO directly — useful in CI or when you need finer control:

```bash
export PLATFORMIO_BUILD_FLAGS="-DFIRMWARE_VERSION='\"v1.16.0-dev\"'"
pio run -e Heltec_v3_repeater
```

Built artefacts land in `.pio/build/<env>/`.

## Flashing

- **ESP32** — use the merged bin with `esptool.py`, the [meshcore.io flasher](https://meshcore.io/flashing.html), or PlatformIO's upload target (`pio run -e <env> -t upload`).
- **nRF52** — drag the `.uf2` onto the mass-storage bootloader drive, or use `nrfutil`.
- **RP2040** — drag the `.uf2` onto the BOOTSEL drive.
- **STM32** — flash the `.bin` or `.hex` with STM32CubeProgrammer or an ST-Link.

!!! tip "Web flasher for first-time installs"
    [meshcore.io/flashing.html](https://meshcore.io/flashing.html) flashes ESP32
    devices directly from a browser — no toolchain required. Use the merged bin
    (`-merged.bin`) for a clean first-time flash that writes the bootloader too.
