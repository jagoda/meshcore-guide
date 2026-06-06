# Boards & HAL

MeshCore runs on a large family of LoRa hardware — nRF52840, ESP32-S3, STM32,
and more — without a single `#ifdef` in the protocol stack. This is achieved
through a layered hardware abstraction:

```
src/MeshCore.h        ← MainBoard (pure virtual interface)
src/helpers/
  ESP32Board.h        ← ESP32 concrete MainBoard implementation
  NRF52Board.{h,cpp}  ← nRF52 concrete MainBoard implementation
arch/
  esp32/              ← radio init, RNG seed, RTCClock for ESP32
  stm32/              ← equivalents for STM32
variants/
  heltec_v3/          ← pin map (target.h) + PlatformIO env (platformio.ini)
  lilygo_t3s3/
  …                   (40+ variants)
```

## `MainBoard` — the hardware interface

`MainBoard` (defined in `src/MeshCore.h`) is the pure-virtual interface every
board must implement:

```cpp
class MainBoard {
public:
  virtual uint16_t getBattMilliVolts() = 0;       // battery ADC
  virtual const char* getManufacturerName() const = 0;
  virtual void reboot() = 0;
  virtual uint8_t getStartupReason() const = 0;   // normal vs wakeup-from-packet
  virtual void sleep(uint32_t secs) { /* no op */ }
  virtual void powerOff() { /* no op */ }
  virtual void onBeforeTransmit() { }              // e.g. light TX LED
  virtual void onAfterTransmit()  { }
  virtual void onBootComplete()   { }
  // … power management, OTA, GPIO, temperature …
};
```

Two startup reasons are defined:

| Constant | Value | Meaning |
|---|---|---|
| `BD_STARTUP_NORMAL` | `0` | Power-on or user reset |
| `BD_STARTUP_RX_PACKET` | `1` | Woke from deep sleep because a LoRa packet arrived |

Applications check `board.getStartupReason()` to decide whether to send an
initial advertisement or skip straight to listening.

## `RTCClock` — the wall-clock interface

```cpp
class RTCClock {
public:
  virtual uint32_t getCurrentTime() = 0;    // UNIX epoch seconds
  virtual void setCurrentTime(uint32_t t) = 0;
  virtual void tick() { }                   // periodic state update (e.g. I²C RTC poll)
  uint32_t getCurrentTimeUnique();          // monotonically increasing, never repeats
};
```

`getCurrentTimeUnique()` ensures timestamps in outbound packets are always
strictly increasing even if the system clock stalls (e.g. GPS cold start).

## `ESP32Board` and `NRF52Board`

These concrete classes implement `MainBoard` for their respective platforms.

### `ESP32Board` highlights

- `sleep(uint32_t secs)`: enters ESP32 light sleep, wakes on timer *or* on
  `P_LORA_DIO_1` GPIO going HIGH (LoRa packet detection). If `inhibit_sleep`
  is set (e.g. during OTA), sleep is skipped.
- `getBattMilliVolts()`: reads `PIN_VBAT_READ` ADC (12-bit), averages 4
  samples, doubles for voltage-divider ratio.
- `onBeforeTransmit()` / `onAfterTransmit()`: drives `P_LORA_TX_LED` or a
  NeoPixel LED if the board defines those pins.
- `ESP32RTCClock`: uses `gettimeofday()` / `settimeofday()` backed by the
  ESP32 RTC peripheral. Initialised to a known-recent time on power-on.

### `NRF52Board` highlights

- `sleep(uint32_t secs)`: calls the Adafruit nRF52 `suspendLoop()` or
  Nordic SYSTEMOFF depending on `NRF52_POWER_MANAGEMENT` compile flag.
- `getBootloaderVersion()` / `startOTAUpdate()`: OTA support via the Adafruit
  nRF52 bootloader (DFU over BLE).
- **Power management extension** (`NRF52_POWER_MANAGEMENT`): voltage-based boot
  protection (refuse to boot below `voltage_bootlock` mV), LPCOMP-triggered
  wake from SYSTEMOFF when battery recovers, shutdown reason codes persisted
  in GPREGRET.

`NRF52BoardDCDC` is a thin subclass that enables the nRF52840's internal
DC/DC regulator on boards that have the required inductors populated.

## `arch/` — platform-level radio glue

The `arch/` directories provide two things that vary per platform but are
shared across all variants of that platform:

1. **`radio_init()`** — initialises the RadioLib driver with the correct pins
   (read from `-D` flags set in the variant's `platformio.ini`), sets LoRa
   parameters (frequency, BW, SF, CR, TX power), and returns `true` on success.
2. **`radio_new_identity()`** — generates a fresh `LocalIdentity` seeded from
   the radio's noise floor (hardware RNG source).

Examples call both in `setup()`:

```cpp
if (!radio_init()) { halt(); }
fast_rng.begin(radio_driver.getRngSeed());
```

## `variants/` — per-target build definitions

Each variant lives in its own subdirectory (`variants/<name>/`) with two files:

### `target.h`

`target.h` is the single header that examples `#include <target.h>`.
It:
- Pulls in the right RadioLib wrapper class (e.g. `CustomSX1262Wrapper`).
- Includes the correct `*Board` header.
- Optionally includes display, sensor, and button helpers.
- Declares the global objects (`board`, `radio_driver`, `rtc_clock`, `sensors`,
  `display`) that `setup()` and `loop()` use.
- Declares `radio_init()` and `radio_new_identity()`.

Example (Heltec Tracker `target.h`):
```cpp
#include <../heltec_v3/HeltecV3Board.h>
#include <helpers/radiolib/CustomSX1262Wrapper.h>
// …
extern HeltecV3Board board;
extern CustomSX1262Wrapper radio_driver;
extern AutoDiscoverRTCClock rtc_clock;
bool radio_init();
mesh::LocalIdentity radio_new_identity();
```

### `platformio.ini`

The variant's `platformio.ini` defines one or more PlatformIO environments
(one per application, e.g. `[heltec_tracker_repeater]`, `[heltec_tracker_companion]`).
Each environment sets:
- `board` — the PlatformIO board JSON (e.g. `esp32-s3-devkitc-1`)
- `build_flags` — pin definitions (`-D P_LORA_CS=…`), platform flags
  (`-D NRF52_PLATFORM`), feature flags (`-D ENABLE_ADVERT_ON_BOOT=1`)
- `build_src_filter` — selects the right `arch/` subdirectory and example source

The root `platformio.ini` pulls all of these in via:
```ini
[platformio]
extra_configs =
  variants/*/platformio.ini
  platformio.local.ini
```

## `boards/` — legacy board JSON

The `boards/` directory contains PlatformIO `.json` board definition files for
boards not yet upstream in the PlatformIO registry (custom MCU configurations,
flash sizes, linker scripts). These are referenced by `board = …` in variant
`platformio.ini` files and are transparently picked up by PlatformIO.

The `boards/` root also contains static JSON board files (e.g.
`heltec_v3.json`). The linker scripts for nRF52 variants (`nrf52840_s140_v7.ld`,
etc.) also live here.

## How it all fits together at build time

```
platformio build -e heltec_tracker_repeater
          │
          ├─ loads variants/heltec_tracker/platformio.ini
          │    → sets board, build_flags (all pin -D macros), build_src_filter
          │
          ├─ compiles examples/simple_repeater/*.cpp
          │    → #include <target.h>  resolves to variants/heltec_tracker/target.h
          │    → target.h pulls in HeltecV3Board, CustomSX1262Wrapper, …
          │
          ├─ compiles src/*.cpp + src/helpers/*.cpp  (the protocol stack)
          │
          └─ links → firmware.elf → .bin / .uf2
```

## Adding a new board — checklist

To bring up MeshCore on a new LoRa board:

1. **Create `variants/<name>/`**.
2. Write `target.h`: include the right `*Board` class and RadioLib wrapper;
   declare the global objects.
3. Write `platformio.ini`: set the platform, framework, board JSON, and all
   pin `-D` macros (`P_LORA_CS`, `P_LORA_BUSY`, `P_LORA_DIO_1`, `USE_SX1262`
   or `USE_SX1276`, `PIN_VBAT_READ` if battery ADC is present, etc.).
4. Add a board JSON to `boards/` if PlatformIO does not already know the MCU.
5. Implement `radio_init()` and `radio_new_identity()` — often by copying the
   nearest `arch/` equivalent and adjusting pin names.
6. Test with `platformio build -e <name>_repeater`; flash; verify serial output.

> **Cross-link:** For step-by-step flash and DFU instructions →
> [Flash Your First Device](../getting-started/flash-your-first-device.md).
> For full board bring-up → [Custom Board Variant](../extending/custom-board-variant.md).
