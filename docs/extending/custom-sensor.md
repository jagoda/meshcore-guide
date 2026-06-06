# Custom Sensor

MeshCore's `simple_sensor` example provides a full, working sensor node. This
cookbook walks you from "I have a hardware sensor" to "its readings appear on the
mesh and can trigger alerts."

## How sensor nodes work

A sensor node is a specialised mesh participant. It:

1. Reads hardware (temperature, humidity, voltage, GPS, …) on a configurable
   interval (default: every 60 seconds, `SENSOR_READ_INTERVAL_SECS`).
2. Encodes readings as [CayenneLPP](https://docs.mydevices.com/docs/lorawan/cayenne-lpp)
   frames — a compact, self-describing binary format.
3. Exposes those readings to authenticated clients via the companion request API.
4. Can fire text alerts to subscribed contacts when a reading crosses a threshold.

The framework that does this lives in `examples/simple_sensor/`:

| File | Role |
|---|---|
| `SensorMesh.h` / `SensorMesh.cpp` | Base class — networking, ACL, CLI, alert machinery |
| `main.cpp` | Concrete subclass (`MyMesh`) + Arduino `setup()`/`loop()` |

## The `SensorManager` interface

Sensor hardware is abstracted behind `src/helpers/SensorManager.h`:

```cpp
class SensorManager {
public:
  double node_lat, node_lon, node_altitude;   // feed these to affect Advert position

  virtual bool begin();
  virtual bool querySensors(uint8_t requester_permissions, CayenneLPP& telemetry);
  virtual void loop();

  // Settings registry (for CLI read/write of sensor config)
  virtual int getNumSettings() const;
  virtual const char* getSettingName(int i) const;
  virtual const char* getSettingValue(int i) const;
  virtual bool setSettingValue(const char* name, const char* value);

  virtual LocationProvider* getLocationProvider();
};
```

The `requester_permissions` bitmask controls what a particular client is allowed
to see:

| Flag | Meaning |
|---|---|
| `TELEM_PERM_BASE` (`0x01`) | Battery voltage (always readable) |
| `TELEM_PERM_LOCATION` (`0x02`) | GPS coordinates |
| `TELEM_PERM_ENVIRONMENT` (`0x04`) | Environment sensors (temperature, humidity, …) |

The built-in `EnvironmentSensorManager` (`src/helpers/sensors/EnvironmentSensorManager.h`)
implements this interface for common I²C sensors (BME280, BME680, soil moisture,
…). For custom hardware you write your own subclass.

## Step 1 — Subclass `SensorManager`

Create `MySensorManager.h` alongside the rest of your application:

```cpp
#pragma once
#include <helpers/SensorManager.h>

class MySensorManager : public SensorManager {
  float _last_co2_ppm = 0.0f;

public:
  bool begin() override {
    // initialise your hardware here
    Wire.begin();
    myCO2Sensor.begin();
    return true;
  }

  bool querySensors(uint8_t perms, CayenneLPP& telemetry) override {
    if (perms & TELEM_PERM_ENVIRONMENT) {
      _last_co2_ppm = myCO2Sensor.readCO2();

      // CayenneLPP channel 2, type LPP_CONCENTRATION (ppm)
      telemetry.addConcentration(2, _last_co2_ppm);
    }
    return true;
  }

  float getLastCO2() const { return _last_co2_ppm; }
};
```

**CayenneLPP channel numbers** — channel 1 is reserved for `TELEM_CHANNEL_SELF`
(the node's own battery voltage). Use channels 2 and up for additional sensors.
Multiple physical sensors on the same channel type use different channel numbers.

**LPP data types** — `LPP_TEMPERATURE`, `LPP_RELATIVE_HUMIDITY`,
`LPP_BAROMETRIC_PRESSURE`, `LPP_VOLTAGE`, `LPP_CONCENTRATION`, `LPP_GPS`, and
more. See the CayenneLPP library headers for the full list.

## Step 2 — Instantiate your manager in `target.cpp`

Every variant exposes a global `sensors` object. Open (or create) your variant's
`target.cpp` and swap in your manager:

```cpp
// target.cpp  (in your variants/<myboard>/ directory)
#include <target.h>
#include "MySensorManager.h"

MySensorManager sensors;   // replaces the default SensorManager
```

`SensorMesh` calls `sensors.begin()` from `setup()` and `sensors.loop()` from
`loop()` automatically once this global is in place.

## Step 3 — Subclass `SensorMesh` and override `onSensorDataRead()`

Open (or create) `main.cpp` in `examples/simple_sensor/` (or your own example
directory). The default `MyMesh` already subclasses `SensorMesh`; extend it:

```cpp
class MyMesh : public SensorMesh {
public:
  MyMesh(mesh::MainBoard& board, mesh::Radio& radio, mesh::MillisecondClock& ms,
         mesh::RNG& rng, mesh::RTCClock& rtc, mesh::MeshTables& tables)
    : SensorMesh(board, radio, ms, rng, rtc, tables),
      co2_data(12 * 24, 5 * 60)   // 24 h of data, sampled every 5 min
  {}

protected:
  Trigger high_co2;
  TimeSeriesData co2_data;

  void onSensorDataRead() override {
    // Cast to our concrete type to access the custom getter
    float co2 = static_cast<MySensorManager&>(sensors).getLastCO2();

    co2_data.recordData(getRTCClock(), co2);
    alertIf(co2 > 1500.0f, high_co2, HIGH_PRI_ALERT, "CO2 level high!");
  }

  int querySeriesData(uint32_t start_secs_ago, uint32_t end_secs_ago,
                      MinMaxAvg dest[], int max_num) override {
    co2_data.calcMinMaxAvg(getRTCClock(), start_secs_ago, end_secs_ago,
                           &dest[0], 2 /* channel */, LPP_CONCENTRATION);
    return 1;
  }

  bool handleCustomCommand(uint32_t ts, char* command, char* reply) override {
    if (strcmp(command, "co2") == 0) {
      float v = static_cast<MySensorManager&>(sensors).getLastCO2();
      sprintf(reply, "CO2: %.0f ppm", v);
      return true;
    }
    return false;
  }
};
```

`onSensorDataRead()` is called every `SENSOR_READ_INTERVAL_SECS` (default: 60 s)
after the `SensorManager::querySensors()` call that populates the CayenneLPP
buffer. Do your alerting and time-series recording here.

`querySeriesData()` is called when an authenticated client requests
`REQ_TYPE_GET_AVG_MIN_MAX`. Return min/max/avg structs for the window requested.

`handleCustomCommand()` lets you add new CLI verbs without touching `SensorMesh`.

## Step 4 — Wire the build

In your variant's `platformio.ini`, add an `[env:...]` entry that points at the
right sources:

```ini
[env:myboard_sensor]
extends = myboard_base
build_flags =
  ${myboard_base.build_flags}
  -D ADVERT_NAME='"My Sensor"'
  -D ADMIN_PASSWORD='"password"'
build_src_filter = ${myboard_base.build_src_filter}
  +<helpers/sensors>
  +<../examples/my_sensor>   ; your main.cpp + MySensorManager.h live here
```

Build:

```bash
export FIRMWARE_VERSION=v0.1.0
sh build.sh build-firmware myboard_sensor
```

## Alert mechanics

`SensorMesh` maintains an alert queue of up to `MAX_CONCURRENT_ALERTS` (4)
`Trigger` objects. Call `alertIf()` from `onSensorDataRead()`:

```cpp
alertIf(bool condition, Trigger& t, AlertPriority pri, const char* text);
```

- When `condition` becomes `true`: the alert is enqueued and sent as a text
  message to each ACL contact that has the matching priority permission
  (`PERM_RECV_ALERTS_HI` or `PERM_RECV_ALERTS_LO`). Up to 4 send attempts are
  made per contact, with ACK tracking.
- When `condition` becomes `false`: the alert clears automatically.

Low-priority alerts make only one send attempt. High-priority alerts make up to
four.

## Reading telemetry from a client

Authenticated companion clients can request telemetry via `REQ_TYPE_GET_TELEMETRY_DATA`.
The sensor node replies with the current CayenneLPP buffer. For historical data,
clients send `REQ_TYPE_GET_AVG_MIN_MAX` with a time window; your `querySeriesData()`
override supplies the response.

!!! note "Permission model"
    The `requester_permissions` byte passed to `querySensors()` comes from the
    ACL entry for the requesting client. Admin clients get `0xFF` (all
    permissions). Use `TELEM_PERM_*` constants to gate sensitive readings.
