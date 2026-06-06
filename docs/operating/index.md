# Operating MeshCore

This section is for people running MeshCore infrastructure — repeaters on rooftops, room servers in sheds, sensor nodes in fields — as well as everyday users who want to get the most out of daily messaging.

The pages here cover the **why** and **how** of real-world operation. Byte-level command specs live on [docs.meshcore.io](https://docs.meshcore.io); this section links out to them where needed rather than restating them.

---

## Pages in this section

| Page | What it covers |
|------|---------------|
| [Daily Use](daily-use.md) | Contacts, direct messages, channels, and node health in the companion app |
| [Running a Repeater](running-a-repeater.md) | End-to-end: flash → configure → tune → monitor a repeater |
| [Running a Room Server](running-a-room-server.md) | BBS post management, joining, admin password, room password |
| [Sensors and Telemetry](sensors-and-telemetry.md) | Deploying sensor nodes, reading telemetry, configuring alerts |
| [Remote Admin CLI](remote-admin-cli.md) | Administering nodes via CLI over the mesh or serial — command reference by category |
| [Power and Battery](power-and-battery.md) | Battery sizing, solar, duty-cycle tuning, nRF52 power management |
| [Troubleshooting](troubleshooting.md) | Deafness, interference, path-hash collisions, GPS, Bluetooth |

---

## Who is this section for?

- **Operators** deploying and maintaining repeaters and room servers.
- **Users** who want to understand path routing, channels, and how to keep their contacts reachable.
- **Tinkerers** building sensor nodes and alert systems.

If you're new to MeshCore, start with [Getting Started](../getting-started/what-is-meshcore.md) and [Core Concepts](../concepts/nodes-and-roles.md) first — this section assumes you understand what a repeater is and how adverts work.

---

**Where the arc continues:** Operators who want to understand *why* the protocol
works the way it does — or who want to build tools and integrations on top of
MeshCore — should continue to [The Protocol →](../protocol/index.md) after
reading this section.
