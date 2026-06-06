# Custom Board Variant

Adding a new hardware board to MeshCore means creating two things: a PlatformIO
board definition (if one doesn't already exist in PlatformIO's registry) and a
**variant directory** that tells the build system how to configure MeshCore for
that board.

## Mental model: boards vs variants

| Layer | What it is | Where it lives |
|---|---|---|
| **Board definition** | MCU, flash/RAM sizes, upload protocol, BSP flags | `boards/<name>.json` |
| **Variant** | MeshCore-specific config: radio pins, I²C pins, feature flags, example apps | `variants/<name>/platformio.ini`, `target.h`, `target.cpp` |

Many boards are already in PlatformIO's own registry. You only need `boards/` if
your MCU is not already known to PlatformIO.

## Step 1 — Board definition (`boards/`)

If your MCU is already supported by PlatformIO (check `pio boards`), skip this
step and reference the existing board name in the variant's `platformio.ini`.

If not, create `boards/<myboard>.json`. Start from the closest existing file in
`boards/` and adjust:

- `"mcu"` — the MCU identifier (e.g. `nrf52840`, `esp32s3`, `rp2040`)
- `"f_cpu"` — CPU frequency in Hz
- `"upload"` — flash size, RAM size, upload protocol
- `"frameworks"` — `["arduino"]`
- `"build"` — extra flags, BSP variant name

The RAK4631 (`boards/rak4631.json`) is a good nRF52840 template. The Heltec v4
(`boards/heltec_v4.json`) is a good ESP32-S3 template.

## Step 2 — Variant directory structure

```
variants/
  myboard/
    platformio.ini    ← build config for this board
    target.h          ← declares board, radio_driver, sensors, rtc_clock globals
    target.cpp        ← instantiates those globals
```

### `platformio.ini`

A minimal variant file has two stanzas: a **board-level base** and one or more
**application environments**:

```ini
; --- Board-level base config ---
[myboard_base]
extends = esp32_base           ; or nrf52_base / rp2040_base / stm32_base
board = myboard                ; matches boards/myboard.json (or PlatformIO registry name)
build_flags =
  ${esp32_base.build_flags}
  -I variants/myboard          ; so #include <target.h> resolves
  -D MY_BOARD_IDENTIFIER=1
  ; LoRa radio: pick your radio class
  -D USE_SX1262
  -D RADIO_CLASS=CustomSX1262
  -D WRAPPER_CLASS=CustomSX1262Wrapper
  ; SPI pins for the radio module
  -D P_LORA_NSS=10
  -D P_LORA_BUSY=13
  -D P_LORA_DIO_1=14
  -D P_LORA_RESET=RADIOLIB_NC
  -D P_LORA_SCLK=9
  -D P_LORA_MISO=11
  -D P_LORA_MOSI=8
  -D LORA_TX_POWER=22
  ; I²C pins (for OLED, sensors)
  -D PIN_BOARD_SDA=17
  -D PIN_BOARD_SCL=18
build_src_filter = ${esp32_base.build_src_filter}
  +<../variants/myboard>       ; compile target.cpp
lib_deps =
  ${esp32_base.lib_deps}

; --- Application environments ---
[env:myboard_repeater]
extends = myboard_base
build_flags =
  ${myboard_base.build_flags}
  -D ADVERT_NAME='"My Board Repeater"'
  -D ADVERT_LAT=0.0
  -D ADVERT_LON=0.0
  -D ADMIN_PASSWORD='"password"'
  -D MAX_NEIGHBOURS=50
build_src_filter = ${myboard_base.build_src_filter}
  +<../examples/simple_repeater>

[env:myboard_companion_radio_usb]
extends = myboard_base
build_flags =
  ${myboard_base.build_flags}
  -D MAX_CONTACTS=350
  -D MAX_GROUP_CHANNELS=40
build_src_filter = ${myboard_base.build_src_filter}
  +<../examples/companion_radio/*.cpp>
lib_deps =
  ${myboard_base.lib_deps}
  densaugeo/base64 @ ~1.4.0
```

The `extends` / `build_src_filter` cascade is intentional — each level adds only
the delta it owns.

### `target.h`

Declares the hardware globals that `main.cpp` and `SensorMesh` use:

```cpp
#pragma once

#include <Arduino.h>
#include <helpers/ArduinoHelpers.h>
#include <helpers/radiolib/RadioLibWrappers.h>   // pick the right wrapper
#include <helpers/SensorManager.h>

// These four globals are referenced by examples and the library core
extern CustomSX1262Wrapper radio_driver;
extern ArduinoMillis        ms_clock;
extern StdRNG               fast_rng;
extern SensorManager        sensors;     // or your subclass
extern AutoDiscoverRTCClock rtc_clock;

extern mesh::MainBoard& board;

bool radio_init();
mesh::LocalIdentity radio_new_identity();
```

### `target.cpp`

Instantiates those globals and implements the radio init helper:

```cpp
#include <target.h>
#include <helpers/ESP32Board.h>    // or NRF52Board, etc.

static ESP32Board _board;
mesh::MainBoard& board = _board;

static SX1262 radio = new Module(P_LORA_NSS, P_LORA_DIO_1,
                                  P_LORA_RESET, P_LORA_BUSY);
CustomSX1262Wrapper radio_driver(radio);

SensorManager sensors;
AutoDiscoverRTCClock rtc_clock;
StdRNG fast_rng;
ArduinoMillis ms_clock;

bool radio_init() {
  SPI.begin(P_LORA_SCLK, P_LORA_MISO, P_LORA_MOSI);
  return radio_driver.begin();
}

mesh::LocalIdentity radio_new_identity() {
  return mesh::LocalIdentity::generateNew(fast_rng);
}
```

Consult `variants/heltec_v3/target.cpp` or `variants/rak4631/target.cpp` for
complete, board-specific examples of display setup, GPIO bring-up, and GPS init.

## Step 3 — Build and iterate

```bash
export FIRMWARE_VERSION=v0.1.0-myboard
sh build.sh build-firmware myboard_repeater
```

On first build, PlatformIO fetches platform packages and library dependencies.
Subsequent builds are fast (incremental).

## Key `build_flags` reference

| Flag | Notes |
|---|---|
| `USE_SX1262` / `USE_SX1276` | Selects the radio driver class |
| `RADIO_CLASS` | C++ class name of the RadioLib radio object |
| `WRAPPER_CLASS` | C++ class name of the MeshCore radio wrapper |
| `P_LORA_NSS`, `P_LORA_BUSY`, `P_LORA_DIO_1` | Radio SPI and interrupt pins |
| `SX126X_DIO2_AS_RF_SWITCH` | Set to `true` if the module uses DIO2 for the TX/RX switch |
| `SX126X_DIO3_TCXO_VOLTAGE` | TCXO reference voltage (common values: `1.8`, `2.4`, `3.3`) |
| `DISPLAY_CLASS` | Display driver class (e.g. `SSD1306Display`, `ST7789Display`) |
| `PIN_BOARD_SDA` / `PIN_BOARD_SCL` | I²C bus for OLED / sensors |
| `NRF52_POWER_MANAGEMENT` | Enables deep-sleep power management on nRF52 |
| `ESP32_CPU_FREQ` | CPU frequency in MHz (80 or 240; 80 saves power) |

## Contributing your variant

If your board is broadly available hardware and your variant is stable, consider
contributing it upstream so others can use it without maintaining a fork. See
[Contributing Upstream](contributing-upstream.md) for the PR process.
