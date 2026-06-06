# Worked Example — meshcore-ha

[`meshcore-ha`](https://github.com/meshcore-dev/meshcore-ha) is an open-source Home Assistant custom integration for MeshCore. It is a real-world Companion API consumer and a useful reference for any Python client targeting the event-driven SDK. This page walks through how it connects, receives messages, fires HA events, and exposes HA services — mapping each piece of the Companion API story to concrete code patterns.

!!! note "Using meshcore-ha?"
    This page is a code walkthrough for developers building their own integrations. For end-user setup and automation recipes, see the [meshcore-ha documentation](https://github.com/meshcore-dev/meshcore-ha/blob/main/README.md).

## Layer 1: meshcore_py SDK

meshcore-ha does not implement the companion protocol wire format itself. It uses [`meshcore_py`](https://github.com/meshcore-dev/meshcore_py), the official Python SDK, which handles transport framing and exposes an **event-driven interface**: you call command methods (`commands.send_appstart()`, `commands.get_msg()`, etc.) and subscribe to typed events via a dispatcher.

This separation lets meshcore-ha focus on Home Assistant integration logic while the SDK handles:

- Transport framing (serial length-prefix, BLE characteristic writes)
- Auto-reconnection (up to `max_reconnect_attempts` retries)
- Contact list caching with a dirty flag (`ensure_contacts(follow=True)`)
- Dispatching typed events to subscribers

## Layer 2: MeshCoreAPI — transport management

`MeshCoreAPI` (`custom_components/meshcore/meshcore_api.py`) wraps the SDK's connection factories and owns the transport lifecycle.

### Connection types

```python
# USB serial
self._mesh_core = await MeshCore.create_serial(
    usb_path, baudrate, auto_reconnect=True, max_reconnect_attempts=100
)

# BLE
self._mesh_core = await MeshCore.create_ble(
    ble_address, auto_reconnect=True, max_reconnect_attempts=100
)

# TCP/Wi-Fi
self._mesh_core = await MeshCore.create_tcp(
    tcp_host, tcp_port, auto_reconnect=True, max_reconnect_attempts=100
)
```

### Startup handshake

After creating the connection, `MeshCoreAPI.connect()` runs the handshake:

```python
# 1. APP_START: identify the app, receive SELF_INFO
appstart_result = await self._mesh_core.commands.send_appstart()
# appstart_result.type == EventType.SELF_INFO on success
self._cache_self_info_event(appstart_result)

# 2. Sync time: set the firmware clock to wall-clock time
await self._mesh_core.commands.set_time(int(time.time()))

# 3. Fire HA connected event for automations
self.hass.bus.async_fire("meshcore_connected", {"connection_type": self.connection_type})
```

After `send_appstart()` succeeds, `SELF_INFO` payload is cached in `_last_self_info` — other components use it to read the node's name and public key without issuing additional commands.

### Disconnect recovery

The SDK's built-in reconnect handles transient drops (up to `max_reconnect_attempts`). When the SDK gives up, it fires an `EventType.DISCONNECTED` event. `MeshCoreAPI` subscribes to this and starts a periodic reconnect task that retries every 60 seconds:

```python
self._mesh_core.dispatcher.subscribe(
    EventType.DISCONNECTED,
    self._handle_disconnect_event
)
```

This two-tier strategy keeps the integration alive across extended outages without blocking HA startup.

## Layer 3: MeshCoreDataUpdateCoordinator — the periodic brain

The `DataUpdateCoordinator` subclass (`coordinator.py`) runs on a periodic timer. Each tick it:

1. Reconnects if `api.connected` is `False`.
2. Calls `commands.get_bat()` to update battery state.
3. Fetches `DEVICE_INFO` on first tick (to learn `max_channels`, firmware version, model).
4. Calls `ensure_contacts(follow=True)` — the SDK returns `True` only if its contact cache changed, avoiding redundant HA entity updates.
5. Drains the message queue on first run (catches anything queued while the integration was loading).
6. After the initial drain, falls back to a **60-second safety-net poll** (`MSG_SAFETY_NET_INTERVAL`) — normal delivery is event-driven (see below), so this only fires after 60 s of silence.

For stats (diagnostics), the coordinator calls `get_stats_core()`, `get_stats_radio()`, and `get_stats_packets()` on a configurable interval when the operator enables the diagnostics feature. These are **local queries only** — they talk to the attached radio, not to the mesh.

## Event-driven message delivery

The key design decision in meshcore-ha is that **messages are delivered by push, not by poll**. This is done by subscribing to `EventType.MESSAGES_WAITING`:

```python
async def handle_messages_waiting(event):
    asyncio.create_task(coordinator.async_flush_messages())

coordinator.api.mesh_core.subscribe(
    EventType.MESSAGES_WAITING,
    handle_messages_waiting
)
```

`async_flush_messages()` calls `commands.get_msg()` in a loop until `EventType.NO_MORE_MSGS`:

```python
async def async_flush_messages(self):
    async with self._message_lock:   # serialize concurrent flush calls
        while True:
            result = await self.api.mesh_core.commands.get_msg()
            if result.type == EventType.NO_MORE_MSGS:
                break
            elif result.type == EventType.ERROR:
                break
            # SDK handles event dispatch; entities subscribed to
            # CHANNEL_MSG / CONTACT_MSG / CHANNEL_DATA_RECV update here
```

The `_message_lock` prevents a race between the `MESSAGES_WAITING` handler and the coordinator's periodic safety-net poll.

## Mapping to HA events

meshcore-ha fires Home Assistant bus events for automations and companion integrations. The primary events are:

| HA event | Trigger |
|---|---|
| `meshcore_message` | Incoming or outgoing channel/DM received |
| `meshcore_delivery_update` | Intermediate RX_LOG delivery detail (progressive) |
| `meshcore_message_sent` | A `send_*` service call completed successfully |
| `meshcore_connected` | Transport connected |
| `meshcore_disconnected` | Transport dropped |
| `meshcore_raw_event` | Every SDK event re-fired verbatim (experimental) |

The `meshcore_message` event schema is documented in full in the [Companion Integration API docs](https://github.com/meshcore-dev/meshcore-ha/blob/main/docs/docs/companion-integration-api.md). The key fields every companion integration needs: `message`, `sender_name`, `pubkey_prefix` (stable cross-event correlation key), `message_type` (`"channel"` or `"direct"`), and `entity_id` (the HA binary sensor for this contact or channel).

## HA services

meshcore-ha exposes services for sending messages and querying device state:

| Service | What it does |
|---|---|
| `meshcore.send_message` | Send a direct message to a contact by `node_id` or `pubkey_prefix` |
| `meshcore.send_channel_message` | Broadcast on a channel by index |
| `meshcore.get_contacts` | Return the device's contact list as a structured response |
| `meshcore.get_channels` | Return configured channel names and secret-presence flags |
| `meshcore.trace` | Path-trace to a contact (hop list, RTT, SNR) |

Internally, `send_channel_message` calls the SDK's channel send command. The SDK fires a `MSG_SENT` event which the coordinator uses to fire `meshcore_message_sent` onto the HA bus.

## HA entities

The coordinator creates HA entities for each contact and channel it discovers:

| Entity pattern | Kind | Represents |
|---|---|---|
| `binary_sensor.meshcore_<device>_<pubkey6>_messages` | binary_sensor | Per-contact last-message activity |
| `binary_sensor.meshcore_<device>_ch_<idx>_messages` | binary_sensor | Per-channel last-message activity |
| `sensor.meshcore_<device>_battery` | sensor | Battery voltage |
| `sensor.meshcore_<device>_signal` | sensor | Last received signal quality |

Contacts discovered passively on the mesh (via advert) but not yet saved to the node's contact table are tracked separately in `_discovered_contacts` and persisted to HA storage across restarts.

## Key takeaways for your own client

meshcore-ha demonstrates several patterns worth copying:

1. **Validate on connect.** Run `send_appstart()` immediately and treat a `None` or error result as a failed connection — don't proceed until the handshake succeeds.
2. **Sync the clock every connect.** Clock drift means messages arrive with wrong timestamps on the mesh.
3. **Push > poll.** Subscribe to `MESSAGES_WAITING` for near-instant delivery; keep the periodic safety-net poll as a fallback only.
4. **Serialise message fetches.** Use a lock to prevent concurrent `get_msg()` calls from racing.
5. **Two-tier reconnect.** Let the SDK auto-reconnect for transient drops; add your own long-cycle retry for extended outages.
6. **Separate entity creation from data fetching.** The coordinator owns data; HA entities subscribe to coordinator updates and the SDK event bus independently.
