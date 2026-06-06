# Sensors and Telemetry

MeshCore's **sensor firmware** turns a LoRa device into an autonomous monitoring node — think weather station, solar-power monitor, GPS tracker, or environmental sensor. Sensor nodes transmit [CayenneLPP](https://docs.mydevices.com/docs/lorawan/cayenne-lpp) telemetry payloads over the mesh, which companion apps and admin clients can query on demand.

---

## What a sensor node does

A sensor node:

- Runs **sensor firmware** (not companion, repeater, or room server firmware).
- Reads one or more physical sensors on a configurable interval (default: every 60 seconds).
- Advertises itself on the mesh so clients can discover and subscribe to it.
- Responds to telemetry query requests from authorised clients.
- Can send **alert messages** to subscribed contacts when a threshold condition is met.

Sensor nodes are accessed through the companion app in the same contact list as repeaters and room servers — the firmware role is `sensor`.

---

## Telemetry data model

Sensor payloads use the **CayenneLPP** binary encoding. Each measurement is a (channel, type, value) triple. Common types used in MeshCore sensor firmware include:

| LPP type | Meaning | Example |
|----------|---------|---------|
| Voltage | Battery or solar input voltage | 3.85 V |
| Current | Charging/load current | 0.42 A |
| Power | Panel or load power | 1.6 W |
| Temperature | Air or board temperature | 23.4 °C |
| Relative Humidity | Ambient humidity | 61% |
| Barometric Pressure | Atmospheric pressure | 1013 hPa |
| Altitude | Elevation above sea level | 320 m |
| GPS | Latitude, longitude, altitude | 45.52°N, 122.67°W, 45 m |

Byte-level encoding details are in the [CayenneLPP spec](https://docs.mydevices.com/docs/lorawan/cayenne-lpp); MeshCore wraps these in its own payload envelope.

### Permission levels for telemetry

The sensor firmware has three telemetry permission levels:

| Level | Permission | What the client sees |
|-------|-----------|---------------------|
| Base | Battery voltage and uptime | Always available to logged-in clients |
| Location | GPS coordinates | Requires `TELEM_PERM_LOCATION` |
| Environment | Temperature, humidity, pressure, etc. | Requires `TELEM_PERM_ENVIRONMENT` |

An admin sets client permissions via the ACL (`setperm` command) to control who can query which sensor data.

---

## Deploying a sensor node — end-to-end

### Step 1 — Flash sensor firmware

1. Connect the device via USB to Chrome at [flasher.meshcore.io](https://flasher.meshcore.io).
2. Select your board and choose **Sensor** firmware (the exact variant depends on what sensors are compiled in for your board).
3. Flash.

### Step 2 — First-time serial configuration

Open the serial console (`picocom -b 115200 /dev/ttyUSB0 --imap lfcrlf` or the web flasher console).

**Frequency (mandatory):**

```
set freq 915.525
```

**Name and location:**

```
set name WeatherMast
set lat 51.5074
set lon -0.1278
```

**Admin password:**

```
password <your-strong-password>
```

The default is `password`. Change it — admin access allows changing configuration and reading private telemetry.

**Flood advert interval** — sensors default to 0 (no automatic flood advert). Set a reasonable interval for autonomous operation:

```
set flood.advert.interval 12
```

This makes the sensor re-announce itself every 12 hours so clients with stale routes can find it again.

**Reboot:**

```
reboot
```

### Step 3 — Confirm on the mesh

Trigger an immediate advert:

```
advert
```

The sensor should now appear in the companion app contact list with role `sensor`. Tap it → **View Telemetry** to pull the current readings.

---

## Reading telemetry in the companion app

1. Open **Contacts** → tap the sensor node.
2. Tap **Telemetry** (or the graph icon).
3. The app sends a `REQ_TYPE_GET_TELEMETRY_DATA` request to the sensor.
4. The sensor replies with the current CayenneLPP payload.
5. The app renders values by type: voltage bars, temperature gauges, a map pin for GPS, etc.

Historical data (min/max/average over a time window) is available via `REQ_TYPE_GET_AVG_MIN_MAX` — the app can chart trends over time if the sensor stores a time-series buffer.

---

## Configuring alerts

Sensor nodes can push unsolicited alert messages to subscribed contacts when conditions are met. The alert system works like this:

1. **Threshold condition** is defined in the sensor application code (e.g. "battery voltage < 3.3 V").
2. When the condition triggers, the sensor sends a direct message to each authorised subscriber.
3. The message retries up to several attempts (8-second ACK window per attempt) to maximise delivery reliability.
4. There are two priority levels — **low** and **high** — which a client can subscribe to independently.

### Setting up alert subscriptions

Subscriptions are managed via the ACL:

```
setperm <client-pubkey> 3        — grant admin (can subscribe to all alerts)
setperm <client-pubkey> 1        — read-only (can query telemetry but not change config)
```

The alert permission flags `PERM_RECV_ALERTS_LO` and `PERM_RECV_ALERTS_HI` are set per client. Check with the sensor operator or the source code for the specific permission bits used in your sensor firmware variant.

---

## Sensor CLI commands

```
sensor list                    — list all sensors and their current values
sensor list <start>            — list from index <start>
sensor get <key>               — read a single sensor value by name
sensor set <key> <value>       — write a sensor setting (e.g. GPS on/off)
gps                            — show GPS status (when GPS compiled in)
gps on / gps off               — enable or disable GPS hardware
gps sync                       — sync device clock from GPS time
gps setloc                     — update stored lat/lon from current GPS fix
gps advert <none|share|prefs>  — control whether GPS location is in adverts
```

Full command reference: [CLI Commands spec](https://docs.meshcore.io/cli_commands/) on docs.meshcore.io.

---

## Power considerations for sensor nodes

Sensor nodes are often deployed remotely on battery or solar. Key settings:

- **`set flood.advert.interval`** — longer interval = fewer TX events = longer battery life.
- **`powersaving on`** — on supported repeater/sensor variants, enables sleep between radio events.
- **Duty cycle** — `set dutycycle <percent>` caps the transmitter's on-air fraction.

nRF52-based sensor hardware (RAK4631, Heltec T114, XIAO nRF52840) supports hardware power management with boot-voltage lockout and LPCOMP wake — see [Power & Battery](power-and-battery.md) and the [nRF52 Power Management spec](https://docs.meshcore.io/nrf52_power_management/) for details.

---

## Troubleshooting sensor nodes

| Symptom | Likely cause | Remedy |
|---------|-------------|--------|
| Sensor not appearing in contacts | Frequency mismatch or no advert | Correct freq; `advert` from serial |
| Telemetry query returns no data | Client not authorised | `setperm <pubkey> 1` (or higher) |
| GPS location stuck at 0,0 | GPS off or no fix yet | `gps on`; wait for fix; `gps setloc` |
| Alert messages not arriving | Client not subscribed or path broken | Check ACL; `advert` to refresh paths |
| Very short battery life | Advert interval too short | Increase `flood.advert.interval` |

For more see [Troubleshooting](troubleshooting.md).
