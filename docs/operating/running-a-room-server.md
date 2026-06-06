# Running a Room Server

A room server is MeshCore's persistent message store — think of it as a bulletin board or mailbox for a group. Users who were offline when a post was made can come back later and receive up to 32 unsynced messages automatically. This makes a room server fundamentally different from a channel: channels are ephemeral (miss it and it's gone); room server posts are stored and replayed.

---

## Room server vs. channel — choose the right tool

| Feature | Channel | Room Server |
|---------|---------|-------------|
| Message persistence | No — receive-only in real time | Yes — up to 32 posts stored |
| Roster | Open to anyone with the key | Users must join (login) |
| Admin control | None | Admin password protects config |
| Guest join | Via shared channel key | Via guest password |
| Repeats flood traffic | Yes | Only if `set repeat on` (not recommended) |

Use a room server when you want a persistent space — a base-camp chat, a neighbourhood notice board, or a group that has members who drop in and out of RF coverage.

---

## End-to-end setup guide

### Step 1 — Flash room server firmware

1. Connect the device via USB to Chrome at [flasher.meshcore.io](https://flasher.meshcore.io).
2. Select your board and choose **Room Server** firmware.
3. Flash.

### Step 2 — First-time serial configuration

Open the console (`picocom -b 115200 /dev/ttyUSB0 --imap lfcrlf` or the web flasher console).

**Set the frequency first:**

```
set freq 915.525
```

**Name your room:**

```
set name Basecamp
```

**Change the admin password:**

```
password <your-strong-password>
```

The default is `password` — change it before putting the server on the air.

**Set the guest (join) password:**

```
set guest.password hello
```

The default guest password is `hello`. You can set this to anything; share it out-of-band with users you want to allow. Leave it blank to create an open room (no password to join).

**Optional — add owner info shown to clients:**

```
set owner.info MeshCore Basecamp | Operated by N0CALL | 915.525 MHz
```

Pipe characters (`|`) become newlines when displayed.

**Optional — set location:**

```
set lat 45.523
set lon -122.676
```

**Reboot:**

```
reboot
```

### Step 3 — Verify the room server appears on the mesh

Trigger a flood advert from the serial console:

```
advert
```

Open the companion app — the room server should now appear in your contact list with role `room`. Tap it; you should be able to join with the guest password.

---

## BBS post management

### How posts flow

1. A user joins the room (authenticates with the guest password).
2. The user sends a post; the room server stores it and **pushes** it to all currently joined users.
3. When a new user (or a user who was offline) joins, the server pushes up to **32 unsynced posts** — the ones the user hasn't acknowledged yet.
4. Posts beyond the 32-slot ring buffer are lost. For a very active room, earlier messages age out.

### Admin controls via CLI

All commands work remotely via the companion app admin CLI (admin password required) or locally via serial.

```
get guest.password              — view current guest password
set guest.password <password>   — change guest password
get allow.read.only             — on/off (default: off)
set allow.read.only on          — let read-only clients join
get acl                         — view the ACL (serial only)
setperm <pubkey> <0|1|2|3>      — set a client's permission level
                                  0=Guest, 1=Read-only, 2=Read-write, 3=Admin
```

**Removing a client from the ACL:**

```
setperm <pubkey>                — omit permissions to remove the entry
```

### Repeating — proceed with caution

A room server can be configured to repeat flood traffic:

```
set repeat on
```

However, the FAQ specifically notes this is **not recommended**: a room server with repeat on lacks the full routing and admin feature set of dedicated repeater firmware. If you need both persistent posts and a repeater, run them on separate devices.

---

## Remote administration via the companion app

Once the room server is in your contact list and you have the admin password:

1. Tap the room server in **Contacts**.
2. Three-dot menu → **Remote Admin** → enter admin password.
3. Type CLI commands in the admin tab.

Most configuration commands that work at the serial console also work remotely. Exceptions (serial-only) are noted in the [CLI Commands spec](https://docs.meshcore.nz/cli_commands/) on docs.meshcore.nz.

---

## Key CLI commands for room server operators

```
ver                          — firmware version
stats-core                   — battery, uptime, queue (serial only)
stats-radio                  — noise floor, RSSI/SNR, airtime (serial only)
get radio                    — current radio parameters
set radio <f>,<bw>,<sf>,<cr> — change radio (reboot to apply)
get name                     — node name
set name <name>              — change node name
clock sync                   — sync clock from connecting companion
advert                       — broadcast flood advert now
reboot                       — restart the node
password <new-pw>            — change admin password
set guest.password <pw>      — change room join password
```

Full reference: [CLI Commands spec](https://docs.meshcore.nz/cli_commands/) on docs.meshcore.nz.

---

## What happens when the room server is unreachable

- **Users who are already joined** will not receive new posts until the server comes back online. The companion app will show delivery failures.
- **New users** cannot join until the server re-adverts and becomes reachable.
- **Posts sent while offline** are not queued by the mesh — if a user sends a post and the room server never receives it, the post is lost. Retry manually once the server is back.
- The room server does **not** have a time-to-live on stored posts; they persist until the 32-slot ring wraps around or the device is erased.

---

## Troubleshooting a room server

| Symptom | Likely cause | Remedy |
|---------|-------------|--------|
| Room server not appearing in contacts | Wrong frequency or no advert received | Set correct freq; `advert` from serial console |
| Can't join as guest | Wrong guest password | `get guest.password` from admin CLI |
| Posts not being pushed | Clock skew between server and client | `clock sync` from admin session |
| Very old last-seen timestamp | Clock not set on server or client | `time <epoch>` via serial; `clock sync` remotely |

For deeper diagnosis see [Troubleshooting](troubleshooting.md).
