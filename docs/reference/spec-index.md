# Spec Index

Curated index of spec and reference pages on [`docs.meshcore.io`](https://docs.meshcore.io). Each entry gives a one-liner, the guide sections that cross-link it, and a direct URL.

This guide's cross-link policy: **we author conceptual narrative and operator workflows; byte-level specs live at `docs.meshcore.io`**. The entries below are the authoritative sources for anything that would change on the next firmware bump.

---

## Packet and payload specs

### [Packet Format](https://docs.meshcore.io/packet_format/)
**What it covers:** Exact byte layout of the header byte, transport codes, `path_length` encoding, path array, and the wire envelope (`dest_hash`, `src_hash`, MAC, ciphertext).

**When to use:** Implementing a parser or serialiser for raw MeshCore packets; understanding how the header byte packs route type, payload type, and version.

**Guide sections that reference this:** [Packet Anatomy](../protocol/packet-anatomy.md), [Packet Journey](../protocol/packet-journey.md), [Encryption on the Wire](../protocol/encryption-on-the-wire.md), [Adverts and Contacts](../concepts/adverts-and-contacts.md)

---

### [Payloads](https://docs.meshcore.io/payloads/)
**What it covers:** Byte-level layout of every payload type (`PAYLOAD_TYPE_*`): advert fields and app-data encoding, text-message inner frame, extended ACK structure, PATH packet encoding, REQ/RESPONSE body conventions, group channel frames.

**When to use:** Building a parser for a specific payload type; verifying advert field offsets; implementing advert signing or verification.

**Guide sections that reference this:** [Payload Types Tour](../protocol/payload-types-tour.md), [Adverts Deep Dive](../protocol/adverts-deep-dive.md), [Adverts and Contacts](../concepts/adverts-and-contacts.md), [Identity and Encryption](../concepts/identity-and-encryption.md)

---

## Companion API specs

### [Companion Protocol](https://docs.meshcore.io/companion_protocol/)
**What it covers:** The complete command and response frame catalogue for the Companion API: every `CMD_*` byte, every `PACKET_*` / `RESP_CODE_*` response type, push notification codes, the startup handshake sequence, and field-by-field payload descriptions.

**When to use:** Building a Companion API client in any language; looking up a specific command code or response field; verifying the startup sequence.

**Guide sections that reference this:** [Frame Model](../companion-api/frame-model.md), [Command Lifecycle](../companion-api/command-lifecycle.md), [Common Operations](../companion-api/common-operations.md), [Build Your Own Client](../companion-api/build-your-own-client.md)

---

### [Stats Binary Frames](https://docs.meshcore.io/stats_binary_frames/)
**What it covers:** Binary encoding of the stats response sub-commands (`CMD_GET_STATS`, sub-types for battery, RSSI, airtime counters, packet counts). Includes the SNR encoding convention (raw value → dB conversion) and all field widths.

**When to use:** Parsing the binary stats payloads returned by a Companion Radio or Repeater; decoding the SNR value from a received packet.

**Guide sections that reference this:** [Common Operations](../companion-api/common-operations.md), [Build Your Own Client](../companion-api/build-your-own-client.md), [Worked Example — meshcore-ha](../companion-api/worked-example-meshcore-ha.md)

---

## CLI and terminal specs

### [CLI Commands](https://docs.meshcore.io/cli_commands/)
**What it covers:** The full command catalogue for the admin CLI available on Repeater and Room Server nodes via USB serial or remote admin. Covers `set`, `get`, `password`, `flood.*`, `advert.*`, and all other settings.

**When to use:** Looking up the exact syntax for a CLI command; building a tool that drives the admin CLI programmatically.

**Guide sections that reference this:** [Remote Admin CLI](../operating/remote-admin-cli.md), [Running a Repeater](../operating/running-a-repeater.md), [Running a Room Server](../operating/running-a-room-server.md)

---

### [Terminal Chat CLI](https://docs.meshcore.io/terminal_chat_cli/)
**What it covers:** The line-by-line command interface for the text terminal client that talks to a Companion Radio node over USB serial. Commands for sending messages, managing contacts, configuring channels.

**When to use:** Using the terminal client directly; scripting terminal-based message sends.

**Guide sections that reference this:** [Remote Admin CLI](../operating/remote-admin-cli.md), [Daily Use](../operating/daily-use.md)

---

## Protocol extensions

### [KISS Modem Protocol](https://docs.meshcore.io/kiss_modem_protocol/)
**What it covers:** The KISS TNC framing used by the `kiss_modem` firmware example to expose raw LoRa packet access to external software (Direwolf, APRSdroid, custom scripts). Frame delimiters, byte-stuffing, and configuration.

**When to use:** Integrating MeshCore with third-party KISS-aware software; implementing a raw packet bridge.

**Guide sections that reference this:** [KISS and Raw Packets](../extending/kiss-and-raw-packets.md)

---

## Number registries

### [Number Allocations](https://docs.meshcore.io/number_allocations/)
**What it covers:** The registry of assigned `PAYLOAD_TYPE_GRP_DATA` data-type values (16-bit): internal-use range `0000–00FF`, registered-app range `0100–FEFF`, and dev/testing range `FF00–FFFF`. How to request a registered data-type range.

**When to use:** Building a custom application on `PAYLOAD_TYPE_GRP_DATA`; registering a data-type range for a published project; choosing a dev-range value during prototyping.

**Guide sections that reference this:** [Custom Application](../extending/custom-application.md), [Glossary](glossary.md)

---

## Hardware and power

### [nRF52 Power Management](https://docs.meshcore.io/nrf52_power_management/)
**What it covers:** nRF52 sleep modes, power rail configuration, current-draw figures for various operating modes, and battery sizing guidelines for nRF52-based boards (RAK4631, etc.).

**When to use:** Optimising battery life for an nRF52-based deployment; understanding why current draw differs between idle, Rx, and Tx modes.

**Guide sections that reference this:** [Power and Battery](../operating/power-and-battery.md)

---

## Sharing and discovery

### [QR Codes](https://docs.meshcore.io/qr_codes/)
**What it covers:** The `meshcore://` URL scheme for contact sharing (`meshcore://contact/add?…`) and channel sharing (`meshcore://channel/add?…`), with parameter definitions and example URLs.

**When to use:** Implementing QR-code generation or import in a custom client; verifying the exact parameter names and encoding for a contact or channel URL.

**Guide sections that reference this:** [QR Codes and Links](qr-and-links.md), [Adverts and Contacts](../concepts/adverts-and-contacts.md), [Channels vs. Direct](../concepts/channels-vs-direct.md)

---

## FAQ / general reference

### [FAQ](https://docs.meshcore.io/faq/)
**What it covers:** Frequently asked questions covering hardware selection, firmware types, common configuration problems, path-hash tuning, GPS, and community resources.

**When to use:** Troubleshooting a deployment problem not covered by this guide's narrative; checking if a question has a canonical answer.

**Guide sections that reference this:** [Troubleshooting](../operating/troubleshooting.md), [What You Need](../getting-started/what-you-need.md)
