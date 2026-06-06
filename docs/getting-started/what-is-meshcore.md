# What Is MeshCore?

MeshCore is an open-source LoRa mesh networking platform for secure, off-grid
text communication — no internet, no cellular, no central server.

---

## LoRa in 60 seconds

**LoRa** (Long Range) is a radio modulation technique that trades data rate for
range and penetration. A single LoRa radio can cover several kilometres in open
terrain; a mesh of nodes can extend that reach further by relaying packets hop
by hop. LoRa operates in unlicensed ISM bands (typically 433 MHz, 868 MHz, or
915 MHz depending on region), so no amateur radio licence is required for
standard use.

The trade-off: bandwidth is in the low-kilobits-per-second range. MeshCore is
built for text messages and compact telemetry, not file transfers or voice.

---

## How a mesh works

In a mesh network, nodes don't just talk to each other directly — they *relay*
messages for each other. A message from Node A to Node C can travel A → B → C,
where B is an intermediate repeater that neither A nor C can directly hear.

MeshCore makes this efficient through two key design choices:

1. **Companion nodes do not repeat.** Your handheld client radio sends and
   receives messages but does not rebroadcast every packet it hears. This keeps
   the mesh quiet and reduces collisions.

2. **Path-based routing after discovery.** The first message to a new contact
   floods the network to discover a path. When the recipient's node acknowledges
   delivery, it sends back a *returned path* — the exact route the message took.
   Subsequent messages follow that path directly, rather than flooding again.
   This is fundamentally different from systems that flood every packet.

---

## The four node roles

| Role | What it does |
|------|-------------|
| **Companion Radio** | Your personal node — connects to a mobile or web app; sends and receives direct messages and channel messages |
| **Repeater** | Relays packets toward their destination; extends network range; does not expose a messaging app |
| **Room Server** | A BBS-style shared message store; users connect and retrieve history even if they were offline when messages were sent |
| **Sensor Node** | Reports telemetry (temperature, GPS position, etc.) to the mesh |

Roles are fixed per-device: a Companion does not repeat, a Repeater does not
connect to a chat app. This separation is intentional — it keeps each node's
behaviour predictable and the channel clean.

---

## How MeshCore compares

=== "vs. Meshtastic"
    Both run on similar LoRa hardware (Heltec, RAK, Lilygo, etc.) and provide
    off-grid text messaging.

    Key differences:

    - **Routing.** Meshtastic floods every packet to every node on the mesh.
      MeshCore floods only the first message to a new contact, then uses the
      returned path for all subsequent messages — significantly less airtime in
      active meshes.
    - **Client role.** In Meshtastic, all nodes (including clients) participate
      in flooding. In MeshCore, Companion nodes never repeat, which keeps
      handheld clients from saturating the channel.
    - **Complexity.** MeshCore's role model requires a bit more thought during
      setup (which firmware type to flash) in exchange for a quieter, more
      efficient mesh.

=== "vs. Reticulum"
    Reticulum is a general-purpose networking layer designed to run over many
    transports, including LoRa via the KISS modem. It has a richer addressing
    model and is well-suited to large, heterogeneous networks.

    MeshCore is more opinionated and purpose-built for off-grid text messaging
    on LoRa hardware: flash a device, connect an app, send a message. If you
    need a programmable network stack that spans multiple link types, explore
    Reticulum; if you want working off-grid messaging in 15 minutes, MeshCore
    is the faster path.

---

## Why MeshCore?

- **No infrastructure required.** No internet. No SIM card. No subscription fee
  for core messaging.
- **Encrypted by default.** Public-key cryptography; direct messages are
  end-to-end encrypted; identity advertised with a signed key to prevent
  spoofing.
- **Open source (MIT).** Firmware is freely auditable and extensible. Anyone
  can build on it without paying anything.
- **Efficient airtime.** Path-based routing after the first flood keeps the
  channel available for everyone.
- **Active ecosystem.** Android, iOS, Windows, Mac, and web clients; a public
  map; Home Assistant integration; community tools for mesh analysis.

---

!!! tip "Next step"
    Now that you know what MeshCore is, [find out what hardware you need →](what-you-need.md)
