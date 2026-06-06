# Example Apps

The six applications in `examples/` are the canonical firmware images.
They are not toy demos — they are the same code that ships on production
hardware. Reading one from `setup()` to `MyMesh` is the fastest way to
understand how the whole stack composes.

## The common pattern

Every example follows the same structure:

```
examples/<name>/
├── main.cpp       ← setup() + loop() wiring
├── MyMesh.h       ← class MyMesh : public <base> (application overrides)
├── MyMesh.cpp     ← override implementations
└── …              ← helper classes (DataStore, UITask, KissModem, …)
```

`main.cpp` is mostly boilerplate: open the filesystem, load or generate the
identity, call `the_mesh.begin()`, then call `the_mesh.loop()` inside
Arduino's `loop()`. The interesting code lives in `MyMesh`.

All six examples include `<target.h>` — the variant-specific header that
provides `board`, `radio_driver`, `rtc_clock`, and `radio_init()` (see
[Boards & HAL](boards-and-hal.md)).

---

## `simple_repeater`

**Role:** A mesh repeater — receives flood packets and re-transmits them,
extending range. Accepts admin CLI commands over serial or from authorised
companion clients.

**Class hierarchy:**
```
Mesh → MyMesh   (uses CommonCLI, ClientACL, RegionMap)
```

**Key mechanics:**
- `allowPacketForward()` override: checks rate limiter, interference
  threshold (from prefs), and loop detection before agreeing to forward.
- `onPeerDataRecv()`: handles admin requests (login, CLI commands, stats
  queries) from authorised companion clients. Replies via `createDatagram`
  sent direct down the return path.
- `onAdvertRecv()`: stores seen neighbours; updates the neighbour table
  (used by `formatNeighborsReply()`).
- Power saving: if `powersaving_enabled` in prefs, calls `board.sleep(30)`
  whenever `the_mesh.hasPendingWork()` is false. On nRF52 this enters
  `suspendLoop()`; on ESP32 it enters light sleep.

**Use as a template for:** Any node that needs to participate in routing but
has no companion-side messaging.

---

## `simple_room_server`

**Role:** A BBS-style room server. Stores and re-broadcasts text messages to
everyone in the room (group channel). Maintains a message log in flash.
Accepts admin CLI commands.

**Class hierarchy:**
```
Mesh → MyMesh   (uses CommonCLI, ClientACL, TxtDataHelpers, group channel)
```

**Key mechanics:**
- `onGroupDataRecv()`: receives channel messages, persists them to a
  flash log file, and re-broadcasts to the channel (so late-joiners receive
  history when they send a discover request).
- `onAnonDataRecv()`: handles anonymous requests for message history.
  Iterates the log and sends each stored message back as a series of
  `PAYLOAD_TYPE_GRP_TXT` packets.
- Room admin: a shared admin password (hashed) gates CLI access. The
  `ClientACL` manages which pubkeys have been granted operator access.

**Use as a template for:** Group coordination nodes — any node that stores
and re-distributes messages to a group.

---

## `simple_sensor`

**Role:** A telemetry sensor node. Periodically reads sensors (battery
voltage, temperature, GPS, …), broadcasts telemetry as CayenneLPP-encoded
data, and sends alert messages when thresholds are exceeded.

**Class hierarchy:**
```
Mesh → SensorMesh (helper) → MyMesh
```

`SensorMesh` (in `src/helpers/`) provides:
- `onSensorDataRead()` virtual hook — override to sample your sensors.
- `alertIf(condition, trigger, priority, message)` — debounced alert sending.
- `querySeriesData()` — time-series query for historical data.
- `TimeSeriesData` — ring-buffer time series store.
- `Trigger` — one-shot / hysteresis alerting logic.

**`MyMesh` in `simple_sensor`:**
```cpp
class MyMesh : public SensorMesh {
  Trigger low_batt, critical_batt;
  TimeSeriesData battery_data;

  void onSensorDataRead() override {
    float v = getVoltage(TELEM_CHANNEL_SELF);
    battery_data.recordData(getRTCClock(), v);
    alertIf(v < 3.4f, critical_batt, HIGH_PRI_ALERT, "Battery is critical!");
    alertIf(v < 3.6f, low_batt,      LOW_PRI_ALERT,  "Battery is low");
  }
};
```

This is the minimal custom sensor: override `onSensorDataRead()`, record data,
call `alertIf()`. Everything else — scheduling, telemetry encoding,
broadcasting, alert deduplication — is handled by `SensorMesh`.

**Use as a template for:** Any node that reads physical sensors and needs to
broadcast or alert on threshold crossings.
See [Custom Sensor](../extending/custom-sensor.md) for a step-by-step guide.

---

## `companion_radio`

**Role:** The Companion radio firmware — the bridge between the mesh and a
companion app (mobile, desktop, or home-automation). Communicates over
BLE, USB serial, or Wi-Fi using the MeshCore frame protocol.

**Class hierarchy:**
```
Mesh → BaseChatMesh → MyMesh
```

`BaseChatMesh` (`src/helpers/BaseChatMesh.h`) adds:
- A contact list (`ContactInfo` array, loaded/saved via `DataStore`).
- An offline queue (messages received while the companion app is disconnected).
- ACK tracking (maps CRCs to pending contacts; drives delivery status).
- Group channel management.

`MyMesh` adds:
- **Frame protocol handler** (`handleCmdFrame()`): parses binary command
  frames from the serial interface (`BaseSerialInterface`) and dispatches to
  mesh operations (send message, get contacts, set channels, etc.).
- **`DataStore`**: persistent contact + channel + prefs storage backed by
  LittleFS / SPIFFS.
- **UI integration** (`AbstractUITask` / `UITask`): optional display and button
  handling for devices with a screen.

The frame protocol is documented at
[`docs.meshcore.nz/companion_protocol`](https://docs.meshcore.nz/companion_protocol).
The companion-API narrative is in
[The Companion API](../companion-api/index.md).

**Use as a template for:** Any node that needs to be controlled by an external
application over a serial/BLE/Wi-Fi API. The companion_radio is also the
starting point for `meshcore-ha` and similar integrations.

---

## `simple_secure_chat`

**Role:** A minimal peer-to-peer secure chat firmware. No companion app
required — the chat interface is over serial. No contact persistence; contacts
are discovered by sniffing advertisements at runtime.

**Class hierarchy:**
```
Mesh → BaseChatMesh → MyMesh
```

Compared to `companion_radio`, `simple_secure_chat` is dramatically stripped
down: no frame protocol, no DataStore, no UI task. The `main.cpp` reads lines
from `Serial`, calls `the_mesh.sendMsg()`, and prints received messages.

**Key constants:**
```cpp
#define PUBLIC_GROUP_PSK  "izOH6cXN6mrJ5e26oRXNcg=="
```
A hardcoded group pre-shared key for an open public channel. All
`simple_secure_chat` nodes share it by default. New contacts discovered via
adverts are added automatically.

**Use as a template for:** The simplest possible chat node; minimal footprint.
Good starting point for learning how `BaseChatMesh` works without the
companion-radio complexity.

---

## `kiss_modem`

**Role:** A KISS TNC (Terminal Node Controller) modem. Presents the LoRa
mesh as a standard KISS interface over serial or UART, allowing any KISS-aware
software (e.g. `Dire Wolf`, `Pat`) to use the mesh as a transport.

**Class hierarchy:**
```
Dispatcher → KissModem   (does NOT extend Mesh)
```

`kiss_modem` is the one example that bypasses `Mesh` entirely. `KissModem`
extends `Dispatcher` directly, implementing `onRecvPacket()` to forward raw
bytes to the KISS serial interface without any payload decoding. Outbound KISS
frames are wrapped in `PAYLOAD_TYPE_RAW_CUSTOM` packets and flooded.

This makes `kiss_modem` the right template for **raw packet** applications —
any use case where you want to send arbitrary byte payloads and handle
your own encryption/protocol above the Dispatcher layer.

> **Cross-link:** KISS frame protocol spec →
> [`docs.meshcore.nz/kiss_modem_protocol`](https://docs.meshcore.nz/kiss_modem_protocol).
> For a KISS integration guide → [KISS and Raw Packets](../extending/kiss-and-raw-packets.md).

---

## Side-by-side summary

| Example | Base class | Admin CLI | Companion API | Key helper used |
|---|---|---|---|---|
| `simple_repeater` | `Mesh` | Yes (serial + remote) | No | `CommonCLI`, `ClientACL`, `RegionMap` |
| `simple_room_server` | `Mesh` | Yes | No | `CommonCLI`, `ClientACL`, group channel |
| `simple_sensor` | `SensorMesh` | Yes (serial) | No | `SensorManager`, `TimeSeriesData` |
| `companion_radio` | `BaseChatMesh` | Limited (rescue mode) | Yes | `BaseSerialInterface`, `DataStore` |
| `simple_secure_chat` | `BaseChatMesh` | No | No | (minimal) |
| `kiss_modem` | `Dispatcher` | No | No | `KissModem` (custom) |

## How to choose a starting point

- Building infrastructure (repeater / room server / sensor)? Start from the
  matching `simple_*` example and add your overrides.
- Building a companion app integration? Start from `companion_radio`; the
  frame protocol and DataStore are already wired.
- Need raw byte transport? Start from `kiss_modem`.
- Want the smallest possible codebase to understand the stack? Read
  `simple_secure_chat` — it has the fewest layers.
