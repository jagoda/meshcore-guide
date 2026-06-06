# Daily Use

This page covers the bread-and-butter operations you'll perform every day as a MeshCore user: managing contacts, exchanging messages, participating in channels, and keeping an eye on node health.

---

## Contacts and the address book

MeshCore nodes discover each other through **adverts** — signed broadcasts that carry a node's name, public key, and optional GPS coordinates.
When your companion radio hears an advert it adds the sender to your contact list automatically.

**To appear on someone else's contact list**, your node must broadcast an advert they can hear.
There are two flavours:

| Mode | What it does | When to use |
|------|-------------|-------------|
| **Zero-hop** | Heard only by nodes within direct RF range | Quick local hello |
| **Flood** | Relayed by every repeater in range (up to 64 hops) | Reaching distant contacts |

Tap the **Advert** button in the companion app to broadcast a flood advert. Your node does *not* advert automatically on a timer by default — you control when you announce yourself.

> **Why not auto-advert?** MeshCore is bandwidth-conservative. Continuous beaconing wastes airtime. Advert when you have something to say or when you need to be found.

### Manually importing a contact

If a node is out of RF range, you can share identity "business cards" out-of-band:

- On the **T-Deck**: `Card to Clipboard` writes `meshcore://…` to the SD card's `clipboard.txt`; `Import from Clipboard` reads it back.
- In the companion app: share the card as text or QR code and import via the QR scanner.

See the [QR & sharing formats reference](../reference/qr-and-links.md) for the exact URL scheme.

---

## Sending direct messages

1. Open your contact list and tap the recipient.
2. Type and send your message.

**How the first message travels:** The first message to any contact floods the mesh.
When the recipient receives it, their node returns a **delivery report** that encodes the path (the sequence of repeaters the message traversed).
From then on, your node embeds that path in every packet — bypassing the flood and dramatically reducing channel load.

**Broken paths:** If a repeater in the stored path goes offline, MeshCore retries the routed message twice, then falls back to flood on the final attempt and resets the stored path. You'll see the retry activity in the app. No manual intervention is needed.

**Resetting a path manually:** If you know the path is stale, you can clear it:
- In the companion app: long-press the contact → **Reset Path**.
- In the terminal chat CLI: `reset path` (while that contact is the active recipient).

---

## Group channels

Channels are shared secrets — anyone with the same secret key can read and send on a channel.
Channel messages **always flood** because there is no single destination to route to.

### The default public channel

A well-known public channel is pre-configured in the companion app:

```
Key (hex):    8b3387e9c5cdea6ac9e5edbaa115cd72
Key (base64): izOH6cXN6mrJ5e26oRXNcg==    (T-Deck entry)
```

The third character is the capital letter **O**, not zero.

### Subscribing to a channel

- **Via QR code:** `meshcore://channel/add?name=<name>&secret=<secret>` — scan with the companion app or T-Deck.
- **Manually:** enter the channel name and shared secret in the app's channel settings.

### Channel vs. direct — when to use which

| Scenario | Use |
|----------|-----|
| Talking to a specific person | Direct message (path-routed, lower airtime) |
| Group conversation, no fixed roster | Channel (flooded, all members receive) |
| BBS-style persistent posts | Room server (see [Running a Room Server](running-a-room-server.md)) |

---

## Node health at a glance

The companion app surfaces a **last-seen timestamp** for each contact. If a repeater or contact shows a very old last-seen:

- **Clock skew** is the most common cause. MeshCore timestamps are absolute (Unix epoch). If your device clock is wrong, contacts appear stale. Sync your clock:
  - In the companion app the clock is set from your phone.
  - For infrastructure nodes: `clock sync` from the admin CLI, or `time <epoch_seconds>` via serial.

- **Out of range / offline.** Trigger an advert; if you get no response the node may be unreachable.

### Terminal chat CLI quick reference

If you're using a USB serial terminal instead of the companion app, the most useful daily-use commands are:

```
advert              — broadcast a flood advert
list [n]            — list contacts (most recent first, optional limit n)
to <name-prefix>    — set the active recipient
send <text>         — send a DM to the active recipient
public <text>       — post to the built-in public channel
reset path          — clear the stored route to the active recipient
clock               — display current device time
ver                 — show firmware version
```

Full command reference: [Terminal Chat CLI spec](https://docs.meshcore.nz/terminal_chat_cli/) on docs.meshcore.nz.

---

## Tips for reliable daily operation

- **Advert after moving.** If you change location significantly, flood-advert so repeaters and contacts update your path.
- **Sync clocks after downtime.** Infrastructure nodes that restart with no GPS or NTP source may drift; `clock sync` from an admin session corrects them.
- **Channels and duty cycle.** If you're on a busy channel in an EU region, check your duty-cycle setting — MeshCore will hold transmissions to stay compliant. See [Airtime & Regions](../concepts/airtime-and-regions.md).
