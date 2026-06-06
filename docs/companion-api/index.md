# Companion API

The Companion API is the binary frame protocol that lets external applications — mobile apps, home-automation integrations, scripts, and custom tools — speak to a **Companion Radio** node over BLE, USB serial, or TCP/Wi-Fi.

This section is a developer narrative guide. Byte-level command and response catalogs live at [docs.meshcore.io](https://docs.meshcore.io/) — the pages here teach the *mental model* you need to read and use those specs confidently.

## What's in this section

| Page | What you'll learn |
|---|---|
| [What It Is](what-it-is.md) | Which firmware runs the API, the three transports, the Companion Radio's gateway role |
| [Frame Model](frame-model.md) | How commands are wrapped for transit and how response frames are identified |
| [Command Lifecycle](command-lifecycle.md) | The request/response rhythm, command queuing, timeouts, push notifications |
| [Common Operations](common-operations.md) | Startup sequence, fetching contacts, sending messages, reading stats |
| [Worked Example — meshcore-ha](worked-example-meshcore-ha.md) | A real Home Assistant integration as an end-to-end walkthrough |
| [Build Your Own Client](build-your-own-client.md) | Transport choices, existing libraries, and gotchas to avoid |

## Quick orientation

Every interaction is a two-layer conversation:

1. **Transport framing** — serialises raw bytes for delivery over BLE characteristics or a UART/TCP stream.
2. **Companion protocol** — the commands and responses your application actually cares about. Each frame starts with a one-byte packet type; the rest is command- or response-specific payload.

You send commands one at a time and wait for the matching response before sending the next. The firmware can also push unsolicited notifications (like "messages are waiting") that arrive outside any command/response pair — your receive loop must handle both patterns.

If you are working in Python or JavaScript, you almost certainly want one of the official SDK libraries rather than raw framing:

- **Python:** [`meshcore_py`](https://github.com/meshcore-dev/meshcore_py)
- **JavaScript/TypeScript:** [`meshcore.js`](https://github.com/meshcore-dev/meshcore.js)

---

**Where the arc continues:** The `meshcore-ha` worked example in this section is
the bridge between *using* the Companion API and *building on* MeshCore. After
working through it, continue to:

- [Architecture & Internals →](../internals/index.md) — to understand the
  firmware internals you are talking to.
- [Extending MeshCore →](../extending/index.md) — to build custom sensors,
  applications, or board variants of your own.
