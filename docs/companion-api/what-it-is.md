# What the Companion API Is

## The Companion Radio firmware

MeshCore hardware can run several different firmware roles — Repeater, Room Server, Sensor, and **Companion Radio**. The Companion API only exists on **Companion Radio** firmware. Repeater and Room Server nodes do not expose this protocol; they communicate via different mechanisms.

A Companion Radio node acts as a **gateway between the LoRa mesh and a connected application**. It handles all the radio-layer complexity — listening for packets, routing, channel encryption, contact management — and exposes a clean command/response interface over a wired or wireless local link. Your application never touches radio packets directly; it sends high-level commands like "send this message to channel 1" and receives decoded, already-decrypted payloads back.

## Transports

The Companion API runs over three transport options. The underlying binary protocol is identical across all three; only the physical link and framing differ.

### Bluetooth Low Energy (BLE)

BLE is the primary transport for mobile apps and single-board computers with Bluetooth support. The node advertises a Nordic UART Service (NUS) compatible service. Your client discovers the service, subscribes to notifications on the TX characteristic (node → app), and writes commands to the RX characteristic (app → node).

- **Service UUID:** `6E400001-B5A3-F393-E0A9-E50E24DCCA9E`
- **RX characteristic** (write to send a command): `6E400002-B5A3-F393-E0A9-E50E24DCCA9E`
- **TX characteristic** (subscribe for responses and notifications): `6E400003-B5A3-F393-E0A9-E50E24DCCA9E`

The default BLE MTU is 23 bytes (20 bytes usable payload). For larger commands such as `SET_CHANNEL`, request an MTU of 512 bytes from the OS and wait for negotiation to complete before sending. See [Frame Model](frame-model.md) for the impact on framing.

Nodes may disconnect after periods of inactivity. Any robust client should implement automatic reconnection with exponential backoff, storing the device MAC address for quick reconnect.

### USB Serial (UART)

USB serial is the most reliable transport for always-connected embedded clients and development desktops. The node's USB port presents as a CDC-ACM serial device. Your application opens the port (typical baud rate: 115 200), then reads and writes framed packets.

The serial transport adds a lightweight length-prefix wrapper around each companion protocol frame — see [Frame Model](frame-model.md#serial-transport-framing).

### TCP / Wi-Fi

Some MeshCore hardware variants expose the Companion API over a TCP socket on a local Wi-Fi network. The session uses the same serial-style length-prefix framing as USB. TCP is useful for home-automation integrations where the radio is mounted remotely from the server.

## The gateway model in practice

When your application connects, the Companion Radio node does not know anything about it yet. The first thing you must do is send `CMD_APP_START` (byte `0x01`) to identify your application and receive the node's self-information in return — radio settings, public key, advertised name, and radio parameters. Without this handshake the node has no confirmation a client is present, and some behaviour (such as manual contact management) requires that the application opt in explicitly.

After the handshake, the node's job is to:

- **Receive** mesh packets, decrypt them, and queue them for your application to pull via `CMD_SYNC_NEXT_MESSAGE`.
- **Transmit** messages your application requests via send commands.
- **Push** notifications like `PUSH_CODE_MSG_WAITING` (`0x83`) so your application knows when to drain the queue without polling on a fixed timer.

Your application's job is to maintain a command queue, drain the message queue promptly when notified, and sync the clock on connection so outbound message timestamps are accurate.
