# KISS and Raw Packets

The KISS modem firmware turns a MeshCore LoRa radio into a standard **KISS TNC**
— a serial (or USB-serial) device that any KISS-capable software can drive. This
lets you plug MeshCore hardware into packet radio software stacks that were never
designed with MeshCore in mind: Direwolf, APRSdroid, YAAC, pat (Winlink),
custom AX.25 tools, and others.

!!! note "Frame-level spec"
    The frame format, SetHardware sub-commands, extension commands, and CSMA
    parameter details are specified in the
    [KISS Modem Protocol](https://docs.meshcore.io/kiss_modem_protocol/) page on
    docs.meshcore.io. This page covers the architecture, use-cases, and how to
    build and use the firmware.

## What the KISS modem does

- Receives raw LoRa packets off the air and delivers them to the host over serial
  as standard KISS Data frames (`0x00`).
- Accepts KISS Data frames from the host and transmits them over LoRa verbatim —
  no MeshCore routing or encryption applied.
- Exposes radio configuration (frequency, bandwidth, spreading factor, TX power,
  RSSI/SNR stats) via the KISS `SetHardware` (`0x06`) command using
  MeshCore-specific sub-commands.

Because it works at the raw packet level, this firmware is appropriate when you
want to:

- Use MeshCore hardware as a general-purpose LoRa TNC.
- Integrate with AX.25 or APRS software that speaks KISS.
- Build a custom protocol that MeshCore's companion API doesn't cover.
- Bridge MeshCore radio hardware into Winlink or other store-and-forward networks.

It is **not** appropriate for applications that need MeshCore's encrypted mesh
routing — those should use a companion radio and the
[Companion API](../companion-api/index.md).

## Serial configuration

Connect at **115200 baud, 8N1, no flow control** — the same settings used by
the companion radio.

On boards that have a dedicated KISS UART (defined by `KISS_UART_RX` /
`KISS_UART_TX` build flags), the KISS traffic runs on `Serial1` and `Serial`
remains free for debug output. On boards without dedicated UART pins, both run
on the same USB-serial port.

## Building the KISS modem firmware

Most supported boards have a pre-defined KISS modem environment. Check with:

```bash
sh build.sh list | grep kiss
```

Typical output:

```
Heltec_v3_kiss_modem
RAK_4631_kiss_modem
LilyGo_T3S3_sx1262_kiss_modem
```

Build and flash:

```bash
export FIRMWARE_VERSION=v1.16.0
sh build.sh build-firmware Heltec_v3_kiss_modem
```

## How it's wired in the source

The implementation lives in `examples/kiss_modem/`:

| File | Role |
|---|---|
| `main.cpp` | Arduino `setup()`/`loop()`; radio init, identity load, KISS dispatch |
| `KissModem.h` / `KissModem.cpp` | KISS frame parser, transmit queue, SetHardware command handler |

`main.cpp` receives raw bytes from the radio driver (`radio_driver.recvRaw()`)
and passes them to `KissModem::onPacketReceived()` — bypassing the MeshCore Mesh
layer entirely. Outgoing frames from the host go through `KissModem::loop()`,
which calls `radio_driver.sendRaw()`.

The `KissModem` class wires up four callbacks you can override from `main.cpp`:

```cpp
modem->setRadioCallback(onSetRadio);        // freq/BW/SF/CR change
modem->setTxPowerCallback(onSetTxPower);    // TX power change
modem->setGetCurrentRssiCallback(onGetCurrentRssi);
modem->setGetStatsCallback(onGetStats);     // rx/tx/error counters
```

This keeps radio hardware access decoupled from the KISS logic — useful if you
port the modem to a new board.

## Connecting from a host application

### Direwolf (Linux/macOS/Windows)

```
# direwolf.conf
CHANNEL 0
MYCALL N0CALL
ADEVICE /dev/ttyUSB0
ARATE 115200
```

Direwolf speaks KISS over serial natively. Point it at the modem's serial port.

### APRSdroid (Android)

Select **TNC (KISS)** as the connection type and set the serial port to the USB
device exposed by the MeshCore board. APRSdroid handles KISS framing automatically.

### Custom Python

The standard `kiss` PyPI package works without modification:

```python
import kiss

k = kiss.KISS('/dev/ttyUSB0', speed=115200)
k.start()

# receive
for frame in k.read():
    print(frame)   # raw bytes, no KISS framing

# send
k.write(my_ax25_bytes)
```

## Raw packet access

If you need even lower-level access — reading and writing LoRa bytes with no
KISS framing — the `recvRaw()` / `sendRaw()` methods on the radio driver wrapper
are exposed for use in custom `main.cpp` implementations. See
`examples/kiss_modem/main.cpp` for the pattern.

## Noise floor calibration and AGC

The KISS modem example runs periodic noise-floor calibration and AGC resets in
`loop()`:

```cpp
// from main.cpp
if ((uint32_t)(millis() - next_noise_floor_calib_ms) >= NOISE_FLOOR_CALIB_INTERVAL_MS) {
  radio_driver.triggerNoiseFloorCalibrate(0);
  next_noise_floor_calib_ms = millis();
}
```

The intervals are `NOISE_FLOOR_CALIB_INTERVAL_MS` (2 s) and
`AGC_RESET_INTERVAL_MS` (30 s). Adjust these constants in `main.cpp` to trade
receive sensitivity against CPU overhead.
