# What You Need

Before you flash firmware, confirm you have the hardware and software covered here.

---

## Hardware — the LoRa device

You need at least **one supported LoRa device**. MeshCore runs on a wide range
of hardware; the definitive, always-current list is on the flasher:

**→ [meshcore.io/flasher](https://meshcore.io/flasher)**

Common entry-level devices that work well:

| Device | Radio chip | Notes |
|--------|-----------|-------|
| Heltec V3 | SX1262 | Popular, affordable; BLE + Wi-Fi; small BLE antenna (keep phone close) |
| RAK WisBlock RAK4631 | SX1262 | Modular design; excellent battery life on nRF52840 |
| Seeed Xiao nRF52 | (SX1262 module) | Compact; good for embedded projects |
| Lilygo T-Deck / T-Deck Plus | SX1262 | Self-contained with keyboard, screen, and optional GPS — no phone needed |
| Seeed T1000-E | SX1262 | Credit-card form factor; built-in GPS |
| Station G2 / G3 | SX1262 (high-power) | Fixed-install repeater or gateway |

!!! note "Check the flasher for the full list"
    New devices are added regularly. This table is illustrative; the flasher at
    [meshcore.io/flasher](https://meshcore.io/flasher) is always current.

### New in v1.17

Eight new board variants, and a new radio family — **LR2021** — alongside the
existing SX1262 / SX1268 / LR1110 / LLCC68 chips:

| Device | Radio chip | Notes |
|--------|-----------|-------|
| Heltec RC32 | SX1262 | |
| Heltec Tower V2 | SX1262 | |
| Heltec V4 R8 | SX1262 | |
| Meshnology W12 | **LR2021** | |
| MeshTracker X1 | **LR2021** | |
| Nibble Zero Connect | SX1262 | |
| ThinkNode M7 | LR1110 | |
| ThinkNode M9 | LR1110 | |

Existing hardware also changed: **Station G3** gains `LoRaFEMControl` — see
[Remote Admin CLI](../operating/remote-admin-cli.md) for the `radio.fem.rxgain`
/ `radio.fem.txgain` commands it enables, and [Boards & HAL](../internals/boards-and-hal.md)
for the `MainBoard` FEM hooks a board opts into.

`cad` (hardware Channel Activity Detection) works on any RadioLib-backed
radio family via the generic `scanChannel()` API, not just LR2021. `extra.sf`
(simultaneous side-detector reception) **is** LR2021-only — it needs multiple
independent detector chains that only the LR2021 exposes.

### Wired backhaul: Ethernet (v1.17)

If you need a fixed-install repeater, room server, or companion radio with a
wired network connection instead of BLE/Wi-Fi, v1.17 adds Ethernet support on
**RAK4631 + RAK13800** (W5100S WisBlock Ethernet module, on a RAK19007 or
RAK19018 base board). See [Connect a Client § Step 2 — Connect to your
radio](connect-a-client.md#step-2-connect-to-your-radio) (Ethernet tab) and
[Remote Admin CLI § Access paths](../operating/remote-admin-cli.md#access-paths)
for the access-path details, and
[FAQ §7.8](https://docs.meshcore.io/faq/) for the upstream hardware
walkthrough.

### How many devices do you need?

| Goal | Minimum devices |
|------|----------------|
| Join an existing mesh near you | 1 |
| Test point-to-point with a friend, no nearby mesh | 2 |
| Build your own mesh with extended range | 3+ (companion + repeater) |

If you only have one device, flash it as **BLE Companion** and connect to the
mobile app — you can start messaging with other MeshCore users in radio range
immediately.

---

## Software — the client app

No compilation needed. Pick the client for your platform:

| Platform | App | Notes |
|----------|-----|-------|
| Android | [Play Store](https://play.google.com/store/apps/details?id=com.liamcottle.meshcore.android) | Free; BLE + USB |
| iOS / iPadOS | [App Store](https://apps.apple.com/us/app/meshcore/id6742354151) | Free; BLE |
| Windows / Mac | [files.liamcottle.net/MeshCore](https://files.liamcottle.net/MeshCore/) | Free; fully unlocked desktop builds |
| Browser (Chrome) | [app.meshcore.nz](https://app.meshcore.nz) | Free; USB Serial |

For **flashing firmware** you'll also need **Google Chrome** (or any
Chromium-based browser) — the web flasher uses the Web Serial API, which
Safari and Firefox do not support.

---

## Regional frequency

LoRa operates in unlicensed ISM bands. The frequency you use must be legal in
your country:

| Region | Frequency band |
|--------|---------------|
| EU / UK | 868 MHz |
| USA / Canada | 915 MHz |
| Australia / New Zealand | 915 MHz |
| Asia (varies by country) | 433 MHz or 915 MHz |

!!! warning "Set your frequency before transmitting"
    A freshly flashed device may default to a frequency that is incorrect for
    your region. The companion app and config tool both offer **regional presets**
    — pick yours during first setup ([covered in Connect a Client](connect-a-client.md)).
    The preset configures frequency, spreading factor, bandwidth, and coding rate
    in one step.

Not sure which preset your community uses? Ask on the
[MeshCore Discord](https://meshcore.gg) for your regional channel.

---

## No internet, no fees, no account

Once firmware is flashed:

- Nodes communicate directly over LoRa radio — no internet path.
- The companion app talks to your local radio over BLE, USB, or Wi-Fi — not
  the internet.
- There is no subscription or registration required for core messaging.

Some **optional** features — deeper map zoom on T-Deck, unlocking the server
remote-management wait gate in the mobile app — are available via small one-time
purchases that support the developers. None of them are needed to flash, connect,
and send your first message.

---

!!! tip "Next step"
    Hardware and browser ready? [Flash your first device →](flash-your-first-device.md)
