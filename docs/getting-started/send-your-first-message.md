# Send Your First Message

Your device is flashed and your app is connected. Time to prove the mesh works.

---

## How contacts work

Before you can send a direct message, your recipient must be in your **contact
list**. Contacts arrive via **adverts** — packets that carry a node's name,
public key, and optional GPS position. When a node advertises and you receive
the advert, that node appears in your contact list.

You can add a contact two ways:

1. **Receive their advert** — wait until their node broadcasts and you hear it.
2. **Import a link or QR code** — share a `meshcore://contact/…` link or QR
   out-of-band (Signal, email, printed QR).

---

## Option A — Add a contact via advert

Best when you and your test contact are within radio range of each other.

1. Open the MeshCore app on **both devices**.
2. On your contact's device, tap **Advert** (sometimes labelled *Announce* or
   *Broadcast*). This transmits their identity over LoRa.
3. If you are within range (directly or via a repeater), their node appears in
   your **Contacts** list within a few seconds.
4. Tap their name to open the conversation.

There are two advert modes — the app may expose both or just one:

| Mode | Range | When to use |
|------|-------|-------------|
| **Zero-hop** | Nodes that directly hear the radio | Both devices are nearby |
| **Flood** | Whole mesh (relayed by repeaters) | Testing through a repeater, or longer range |

!!! tip "Not appearing? Try flood"
    If your contact doesn't show up after a zero-hop advert, ask them to send
    a **flood advert** so repeaters relay it across the mesh.

---

## Option B — Add a contact via QR / link

Best when your test contact isn't in radio range yet, or when you want to
pre-load contacts before going off-grid.

1. On your contact's device, go to **Identity** → **Share** → **Copy Link** or
   **Show QR code**.
2. In your app, scan the QR code or paste the `meshcore://contact/…` link.
3. They appear in your contact list immediately — no radio exchange needed to
   *add* them (but you will need radio range to actually *message* them).

---

## Send a direct message

1. In your **Contacts** list, tap the contact you want to message.
2. Type your message and tap **Send**.
3. On the first send to this contact, MeshCore floods the message to discover
   a path to the recipient.
4. When the recipient's node gets the message, it sends back a delivery
   acknowledgement (ACK) that also carries the path the message took.
5. Your app shows a confirmation when the ACK arrives — the path is now stored
   for future messages.

**Subsequent messages** to the same contact follow the discovered path directly,
without flooding — much more efficient.

### Delivery status

| Status | What it means |
|--------|--------------|
| Sending… | Packet transmitted; waiting for ACK |
| ✓ Delivered | Recipient's node acknowledged; message arrived |
| ✗ Failed (retrying) | No ACK yet; MeshCore will retry |
| ✗ Failed | No ACK after all retries; node unreachable on current path |

On the final retry, MeshCore falls back to flooding to try to re-establish a
path. If the destination is reachable at all, this usually succeeds and the path
is updated for next time.

---

## Join a channel (group messaging)

You can also message a group via a shared **channel**. All nodes that know the
channel's shared secret can participate.

1. In the app, tap **Channels** and then **Add channel**.
2. Enter the channel name and shared secret — or import a
   `meshcore://channel/…` link or QR code.
3. Type a message and tap **Send** — it is broadcast to all nodes sharing that
   secret.

For testing with another nearby node, you can both join the default public
channel using this shared secret:

```
Hex:    8b3387e9c5cdea6ac9e5edbaa115cd72
Base64: izOH6cXN6mrJ5e26oRXNcg==
```

!!! note "Channel messages always flood"
    Unlike direct messages, channel messages have no fixed path — they flood
    to every node in range that shares the channel secret. This uses more
    airtime than direct messages; keep channel chatter proportionate.

---

## Congratulations

You have flashed firmware, connected a client, and sent your first mesh message.
That is the core loop — everything else builds on this foundation.

**Where to go next:**

- [Core Concepts](../concepts/index.md) — understand *why* the mesh behaves
  as it does: roles, adverts, routing, encryption
- [Running a Repeater](../operating/running-a-repeater.md) — add a second device
  to extend your network's range
- [MeshCore Discord](https://meshcore.gg) — find your regional mesh community
- [MeshCore Map](https://map.meshcore.io) — see who is on the air near you
