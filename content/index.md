---
title: Project Rhythm Core
---

Technical documentation for **Project Rhythm Core** — a rhythm game engine built around
one hard requirement: knowing, to the millisecond, which audio sample is playing right now.

## Components

- **[[audio-engine/index|Audio Engine]]** — the timing backbone. A native Rust audio
  module (cpal + Symphonia) driven from Electron, a playback clock anchored to audible
  time, and a renderer-side clock that costs nothing per frame.

## Repositories

| Repository | Contents |
| --- | --- |
| `Game-Client` | The game client: Electron shell, Svelte renderer, and `rust-core`, the native engine |
| `Docs` | This documentation |

Each component is documented as it is actually built: what it does, why it is built that
way, and the exact surface it exposes to the rest of the application.
