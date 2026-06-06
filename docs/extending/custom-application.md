# Custom Application

MeshCore's standard payload types cover messaging, telemetry, routing, and control.
When you need something entirely different — a sensor protocol, a game state
vector, a custom data feed — you can introduce a new **group data payload** with
your own 16-bit data-type identifier.

This cookbook covers the concept; byte-level frame layout lives in the
[Payload Types spec](https://docs.meshcore.io/payloads/) on docs.meshcore.io.

## How group data payloads work

`PAYLOAD_TYPE_GRP_DATA` packets carry an arbitrary binary payload prefixed with a
16-bit **data-type** field. Any node that doesn't recognise the data-type simply
drops the payload — so different applications share the mesh without interfering.

The 16-bit space is partitioned:

| Range | Purpose |
|---|---|
| `0x0000–0x00FF` | Reserved — internal MeshCore use |
| `0x0100–0xFEFF` | Allocated custom applications (registered in the number-allocations table) |
| `0xFF00–0xFFFF` | Development / testing — use freely, no registration required |

For the full allocation table and registration instructions, see the
[Number Allocations](https://docs.meshcore.io/number_allocations/) page on
docs.meshcore.io.

## Development workflow

**Phase 1 — build and test with a dev data-type**

Pick any data-type in the `0xFF00–0xFFFF` range. This range is explicitly
reserved so multiple projects can coexist during development without allocating
anything official.

```cpp
// myapp.h
#define MY_APP_DATA_TYPE   0xFF42   // dev/testing range — no registration needed
```

Build your sender and receiver logic, iterate, test end-to-end.

**Phase 2 — register before publishing**

Once your application is working and you're ready to ship, open a pull request
against the MeshCore repository that adds a row to
`docs/number_allocations.md`:

```markdown
| 0x0142            | My Application              | you@example.com — https://github.com/you/myapp |
```

The maintainers will review that your app exists, works, and doesn't conflict with
existing allocations. After merge you have an official data-type range and can
update your firmware constants.

## Sending a group data packet

Your application subclasses `mesh::Mesh` (or one of the example base classes like
`SensorMesh`, `BaseChatMesh`). Use `createGroupDataPacket()` to build the packet
and `sendFlood()` or `sendDirect()` to transmit:

```cpp
#include <Mesh.h>

// Within your Mesh subclass:
void sendMyAppData(const uint8_t* payload, size_t len) {
  // Build a group-data packet for MY_APP_DATA_TYPE
  uint8_t buf[MAX_PACKET_PAYLOAD];
  uint16_t dtype = MY_APP_DATA_TYPE;
  buf[0] = (dtype >> 8) & 0xFF;
  buf[1] =  dtype       & 0xFF;
  memcpy(&buf[2], payload, len);

  auto pkt = createGroupData(buf, 2 + len);
  if (pkt) sendFlood(pkt, 0, 1);
}
```

!!! note "Channel scope"
    Group data packets are scoped to the current channel key (region/scope).
    Nodes not sharing that channel key will not decrypt the payload. If you want
    unauthenticated broadcast behaviour, use the default (global) scope.

## Receiving a group data packet

Override `onGroupDataRecv()` in your `Mesh` subclass:

```cpp
void onGroupDataRecv(mesh::Packet* packet, uint8_t* data, size_t len) override {
  if (len < 2) return;
  uint16_t dtype = ((uint16_t)data[0] << 8) | data[1];

  if (dtype != MY_APP_DATA_TYPE) return;   // not for us

  // process data[2 .. len-1]
  handleMyAppPayload(&data[2], len - 2);
}
```

Drop anything that isn't your data-type. The mesh delivers group-data packets to
every node on the channel, so filtering early keeps your application isolated from
other apps sharing the channel.

## Payload design tips

- **Keep payloads compact.** MeshCore's maximum packet payload (`MAX_PACKET_PAYLOAD`)
  is 256 bytes including the data-type prefix, encryption MAC, and routing path.
  Aim for ≤ 200 bytes of application data.
- **Include a version byte.** Even for a private application, a version field in
  the payload lets you evolve the schema without replacing the data-type.
- **Use CayenneLPP if your data is telemetry.** The sensor framework already knows
  how to encode/decode it, and companion apps can display it out-of-the-box.
- **Plan for replay.** The mesh may deliver a packet more than once. Include a
  sequence number or timestamp so receivers can detect duplicates.

## Relationship to the companion API

The companion API (serial/BLE frame protocol) is separate from the mesh payload
system. If you want your application's data to be readable by a companion app:

1. The companion radio firmware needs to forward the group-data packets to the
   connected host over the companion serial protocol.
2. The host app then interprets the forwarded payload.

See the [Companion API](../companion-api/index.md) section for the frame-level
protocol, and the [Number Allocations](https://docs.meshcore.io/number_allocations/)
page for the companion-side data-type registry.
