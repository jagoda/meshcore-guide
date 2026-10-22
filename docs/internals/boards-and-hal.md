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
  …                   (90+ variants)
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
  virtual bool handleCommand(const char* cmd, uint32_t ts, char* reply) { return false; }  // v1.18
  virtual void loop() { /* no op */ }              // v1.18: watchdog / housekeeping tick
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

### Board hooks added in v1.18

Three things were added to `MainBoard` (`src/MeshCore.h`) in v1.18, all with
default bodies so existing boards compile unchanged:

| Hook | Purpose |
|---|---|
| `handleCommand(cmd, ts, reply)` | Per-board CLI verbs. `CommonCLI` tries the radio commands first, then calls this before answering "Unknown command" — the T-Beam 1W uses it for `get fan` / `set fan`, Heltec boards for `radio.fem.*` |
| `loop()` | Called from every example's `main.cpp` `loop()`; lets a board feed its watchdog and run periodic housekeeping |
| `attachDynamicPrefs(KeyValueStore*)` | Hands the board the node's key/value preference store so variant-specific settings (`fan`, `fan_lo`, `fan_hi`, `fem_rxgain`, `fem_txgain`) persist across reboots |

Power-source reporting also grew in v1.18: `isChargerActive()`, plus the
optional `hasUsbPowerDetect()` / `isUsbPowered()` / `hasSolarChargerDetect()` /
`isSolarChargerActive()` set, which the CLI's `power` command reports as
`usb:` / `solar_chg:` (or `n/a` when a board does not override them).

### FEM (front-end module) control — v1.17 hooks, replaced in v1.18

v1.17 put six FEM virtuals on `MainBoard` (`setLoRaFemLnaEnabled()`,
`canControlLoRaFemLna()`, `isLoRaFemLnaEnabled()` and the `…PaGain…` trio) so a
board with an external LNA/PA could be driven from `CommonCLI`.

**v1.18 removed all six from `MainBoard`.** FEM support is now board-local, in
the same `attachDynamicPrefs()` + `handleCommand()` pair described above: each
board owns its `radio.fem.*` verbs and reads/writes the state through the prefs
store, so the setting survives a reboot. Boards that implement it at v1.18.0:
Heltec Tracker V2, Heltec V4, Heltec V4 R8 and Heltec T096 expose
`get/set radio.fem.rxgain` (LNA); the new WIO Tracker L1 1W exposes
`get/set radio.fem.txgain` (it switches the 1 W PA supply through
`SX126X_POWER_EN`) and answers `Error: unsupported` for `rxgain`. Station G3
still implements both LNA and PA-gain control, but as its own `LoRaFEMControl`
methods rather than `MainBoard` overrides. A board that implements neither
answers `Unknown command`. See [Remote Admin CLI § Radio
configuration](../operating/remote-admin-cli.md) for the CLI-level behaviour.

For the record, the v1.17 hook set was:

```cpp
// v1.17 only — removed from src/MeshCore.h in v1.18
virtual bool setLoRaFemLnaEnabled(bool enable) { return false; }
virtual bool canControlLoRaFemLna() const { return false; }
virtual bool isLoRaFemLnaEnabled() const { return false; }
virtual bool setLoRaFemPaGainEnabled(bool enable) { return false; }
virtual bool canControlLoRaFemPaGain() const { return false; }
virtual bool isLoRaFemPaGainEnabled() const { return false; }
```

All six defaulted to `false`, so a board **opted in** by overriding them; in
v1.17 `CommonCLI` checked `canControlLoRaFem{Lna,PaGain}()` before honouring
`radio.fem.rxgain` / `radio.fem.txgain`. v1.18's per-board `handleCommand()`
replaces that guard: the board answers the verb itself, so a board without FEM
support returns "Unknown command" rather than `Error: unsupported`.

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
  ESP32 RTC peripheral. As of v1.18 the clock also survives resets: the current
  time is mirrored into `RTC_NOINIT_ATTR` backup words with a magic value
  (`RTC_BACKUP_MAGIC`) and restored on any reset that is not deep sleep, so a
  crash, watchdog or brownout no longer rewinds the clock to a hard-coded seed
  date (`RTC_TIME_MIN`, 1 Mar 2026 — it was 15 May 2024 before v1.18). Full
  power loss still loses it. This is **ESP32-only**; nRF52/STM32/RP2040 boards
  have no such backup and restart at their seed time unless they have an I²C
  RTC or GPS.

### `NRF52Board` highlights

- `sleep(uint32_t secs)`: calls the Adafruit nRF52 `suspendLoop()` or
  Nordic SYSTEMOFF depending on `NRF52_POWER_MANAGEMENT` compile flag.
- `getBootloaderVersion()` / `startOTAUpdate()`: OTA support via the Adafruit
  nRF52 bootloader (DFU over BLE).
- **Power management extension** (`NRF52_POWER_MANAGEMENT`): voltage-based boot
  protection (refuse to boot below `voltage_bootlock` mV), LPCOMP-triggered
  wake from SYSTEMOFF when battery recovers, shutdown reason codes persisted
  in GPREGRET.

!!! note "v1.18: power management is no longer compiled in on most nRF52 boards"
    Everything described in the bullet above — `checkBootVoltage()`,
    `enterSystemOff()`, `configureVoltageWake()` — sits inside
    `#ifdef NRF52_POWER_MANAGEMENT`, which each variant opts into with
    `-D NRF52_POWER_MANAGEMENT` in its `platformio.ini`. Thirteen nRF52
    variants set it in v1.17.1; at v1.18.0 only three do: `heltec_t1`,
    `heltec_tower_v2` and `muzi_base`, all at `PWRMGT_VOLTAGE_BOOTLOCK` =
    3100 mV. On every other nRF52 board the block is not compiled at all, so
    `config->voltage_bootlock` is never read — even though many of those
    `variant.h` files still carry the `#define`. The CLI matches the gating:
    `get pwrmgt.support` answers `unsupported` and `pwrmgt.source` /
    `pwrmgt.bootmv` return `ERROR: Power management not supported`, while
    `get pwrmgt.bootreason` is not compiled out and reports `Not available`.
    v1.18 also renamed the private `initPowerMgr()` to an idempotent
    `pwrmgtInit()` and made LPCOMP wake arming depend on
    `isPwrMgtInitialised()` and `getWakeLpcompSupported()`.

    For the operator-facing side — per-board table, thresholds, upgrade
    history, what to do about a node that can now boot on a flat cell — see
    [Power & Battery § nRF52 power
    management](../operating/power-and-battery.md#nrf52-power-management-hardware-level).

`NRF52BoardDCDC` is a thin subclass that enables the nRF52840's internal
DC/DC regulator on boards that have the required inductors populated.

## `MultiSerialInterface` — four transports at once (v1.17)

`src/helpers/MultiSerialInterface.h` implements `BaseSerialInterface` by
multiplexing up to **4** concrete transports (`MAX_INTERFACES`) behind one
interface: `InterfaceType::Bluetooth`, `USB`, `WiFi`, and (new in v1.17)
`Ethernet`. A board's `MyMesh` calls `addInterface(type, instance)` for each
transport it supports; `MultiSerialInterface` fans frame reads/writes out to
whichever registered interface is active, so the companion frame protocol
itself is transport-agnostic — the same `handleCmdFrame()` code path serves a
BLE client, a USB client, or (on Ethernet-flashed RAK4631 hardware) a TCP
client on port 5000, without transport-specific branches in the application
layer. This is the same mechanism that lets Ethernet **replace** BLE on
companion Ethernet firmware rather than needing to run alongside it — see
[Remote Admin CLI § Ethernet (TCP)](../operating/remote-admin-cli.md#ethernet-tcp-v117).

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

### Variants added in v1.18

| Variant dir | Board | MCU | Radio | Notes |
|---|---|---|---|---|
| `thinknode_m8` | ThinkNode M8 | nRF52840 | SX1262 (`CustomSX1262Wrapper`) | 1.54" e-ink (`GxEPD2_154_D67`), rotary + nav input, TCXO 3.3 V, 22 dBm |
| `thinknode_m4` | ThinkNode M4 | nRF52840 | LR1110 (`CustomLR1110Wrapper`) | `RF_SWITCH_TABLE`, TCXO 1.6 V, GPS (`ENV_INCLUDE_GPS`), 22 dBm |
| `waveshare_esp32s3_lr1121` | Waveshare ESP32-S3 LR1121 | ESP32-S3FH4R2 (4 MB flash / 2 MB PSRAM) | LR1121 (`CustomLR1121` / `CustomLR1121Wrapper`) | 850–930 MHz **and** 2.4 GHz; `LR1121_FORCE_HP_PA` (LP PA unpopulated); native USB-CDC; no LED, battery sense or user button |
| `muzi_base` | Muzi Base (MuziWorks) | nRF52840 | Duo = LR1121, Uno = SX1262 | one variant dir, radio picked per environment (`muzi_base_duo_*` / `muzi_base_uno_*`); the `superIO` environments add a SH1107 display, joystick, RX8130CE RTC, GPS and buzzer |
| `wio-tracker-l1-1w` | Seeed WIO Tracker L1 1W | nRF52840 | SX1262 + 1 W PA | SH1106 OLED, GPS at 9600 baud, PA supply switched via `SX126X_POWER_EN` (exposed as `radio.fem.txgain`) |

Three new PlatformIO board JSONs came with them: `boards/thinknode_m8.json`,
`boards/thinknode_m4.json` and `boards/muzi_base.json`.

### Board and radio support changes in v1.18

| Area | Change |
|---|---|
| LR1121 (new driver) | `CustomLR1121` / `CustomLR1121Wrapper`: switches the LR1121 to its DC/DC regulator after `begin()`, clips TX power to the active PA's range (the HF PA tops out at 13 dBm) instead of failing, re-applies the configured power after a frequency change, and recovers RX after a header error. Used by `waveshare_esp32s3_lr1121` and Muzi Base Duo |
| LR1110 | `CustomLR1110::begin()` now also enables the DC/DC regulator (RadioLib defaults to LDO) |
| LR2021 | `RadioLibWrapper::setTxPower()` takes the radio `idle()` before `setOutputPower()` under `USE_LR2021`. The `startReceive()` change in `CustomLR2021.h` between v1.17.1 and v1.18.0 is whitespace only |
| RAK3401 GPS | GPS now initialises on this board: the RAK12500-on-I²C probe no longer toggles `WB_IO2` (`#ifndef RAK_3401` — that rail also feeds the PA), and `PIN_GPS_TX` / `PIN_GPS_RX` were swapped onto the right pins (they were crossed in v1.17.1) |
| GPS detect | Cold-start modules are polled for up to ~5 s (20 × 250 ms) instead of a single check after a fixed 1 s delay, so slow modules are no longer missed at boot |
| Bosch env sensors | BME680, BME680+BSEC, BME280 and BMP280 are now probed at **both** 0x76 and 0x77 (SDO strap), with the alternate treated as a fallback rather than a second device |
| ProMicro | INA226 current/voltage sensor support (`ENV_INCLUDE_INA226`, `robtillaart/INA226`) |
| T-Echo Lite | New `LilyGo_T-Echo-Lite_companion_radio_usb` environment (USB-CDC companion). The KISS environment itself is unchanged from v1.17.1, only moved in the file |
| Heltec CT62 | `board_build.partitions = min_spiffs.csv` to fit around the 4 MB flash limit |
| T-Beam 1W | Real fan control: NTC-based auto mode (on at 36 °C, off below 30 °C), 15 s TX cooldown, fail-safe on a bad NTC reading, plus `get fan` / `set fan` / `set fan.lo` / `set fan.hi` via the board `handleCommand()` hook, persisted in the prefs store |
| T1000-E | The `SENSOR_EN` gate (P0.04) was removed — temp/lux sensors now run off `PIN_3V3_EN` alone; GPS sleep/reset handling moved into `src/helpers/sensors/AirohaSleep.h` with corrected `GPS_VRTC_EN` / `GPS_SLEEP_INT` polarity |
| Heltec T1 | `DISABLE_DS3231_PROBE` — the onboard ICM-42607-P IMU answers at I²C 0x68, the same address the DS3231 probe checks, which produced a bogus, unfixable clock |
| Meshnology W12 | TX power defaults raised: `LORA_TX_POWER` 4 → 5 dBm (~22 dBm at the antenna) and `MAX_LORA_TX_POWER` 4 → 13 dBm (measured ~28 dBm); its companion build also gained `ENABLE_WIFI_INTERFACE` |
| WIO E5 (dev/mini) | **Could not verify.** No change to `variants/wio-e5-dev` or `variants/wio-e5-mini` between v1.17.1 and v1.18.0, and no WIO-E5-specific TX-power change elsewhere in the tree |

RAK12501 is not referenced anywhere in the v1.18.0 tree; only the RAK12500
(I²C u-blox) path exists, so RAK12501 support could not be confirmed.

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
