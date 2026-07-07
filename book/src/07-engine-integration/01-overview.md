# Overview

The Platform API is engine-agnostic. The **Engine Integration** layer defines
how specific engines work alongside it — where the boundary is, how a
rendering engine and the Platform API coexist in one application, and how to
provide engine-specific helpers and starter kits without coupling the
platform to any engine.

## The boundary

```text
Application
    ├── Engine (rendering, scene, physics, audio)  ← developer's choice
    └── PlayOS Platform API (platform services)     ← this specification
```

The engine renders frames; the Platform API reports input, stores save data,
discovers capabilities, and handles lifecycle. Neither includes the other.

## What "engine-agnostic" means in practice

- The Platform API does not know which engine is calling it.
- An engine integration kit (for example the Raylib Reference Kit) provides
  convenience helpers, not platform behavior.
- A developer using SDL, Godot, or a custom engine writes the same
  `PlayOS::Input::Pressed(Button::A)` call. Only the rendering setup differs.

## Engine integration kits

A kit is a lightweight layer in a PlayOS repository (for example
`playos-platform-api` or a companion repo) that provides:

- **Build integration** — CMake snippets to link the Platform API alongside
  the engine.
- **Input bridging** — a way to feed engine input events into logical
  PlayOS buttons and axes (and vice versa).
- **Lifecycle helpers** — wiring the engine's frame loop to
  `Lifecycle::Update`.
- **Samples and templates** — starter projects showing the two working
  together.

The kits do **not** change the Platform API. They make it easier to adopt.

## Reference engines

| Engine | Kit | Status |
|---|---|---|
| Raylib | [Raylib Reference Kit](03-raylib-reference-kit.md) | ✅ Active (shell + samples use it) |
| SDL | [SDL Integration](04-sdl-integration.md) | Planned |
| Godot | [Godot Integration Future](06-godot-integration-future.md) | Future |
| Custom | [Custom Engine Integration](05-custom-engine-integration.md) | Guidelines |

See also the [Engine-Agnostic Contract](02-engine-agnostic-contract.md) for
the formal rules an engine must follow to coexist with the Platform API.
