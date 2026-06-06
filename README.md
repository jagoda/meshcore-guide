# MeshCore Guide

Beginner-to-advanced narrative guide for [MeshCore](https://github.com/ripplebiz/MeshCore) —
a LoRa mesh networking stack.

**Byte-level specs** (packet format, payload tables, companion protocol frames, CLI commands)
live at [docs.meshcore.nz](https://docs.meshcore.nz). This guide teaches the mental models,
operator workflows, and developer architecture that make those specs meaningful.

## Reading tracks

**Operator** → Getting Started → Core Concepts → Operating MeshCore

**Developer** → Getting Started → Core Concepts → The Protocol → Companion API → Architecture & Internals → Extending MeshCore

## Building locally

Requires Python 3.11+ and [uv](https://github.com/astral-sh/uv).

```bash
uv sync
uv run mkdocs serve
```

Open <http://127.0.0.1:8000> to preview the site.

## Status

This site is under active development. Pages marked `<!-- TODO -->` are stubs
awaiting authored content.

## License

Content: [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/)
