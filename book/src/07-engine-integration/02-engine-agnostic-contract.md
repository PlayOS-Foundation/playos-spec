# Engine-Agnostic Contract

The Platform API is **engine-agnostic** — it does not know, depend on, or
care which engine is rendering frames. This chapter defines the contract
that makes that possible.

## The rule

> The Platform API MUST NOT `#include` any game-engine header. An engine
> MUST NOT provide Platform API services.

Each side stays in its lane:

```text
Engine                              Platform API
─────────────────────────           ─────────────────────────
Window creation                     Button input (logical)
Rendering (2D/3D)                   Axis input (analog)
Scene graph / ECS                   Save data paths
Physics                             Capability discovery
Audio playback / mixing             Master volume control
Asset loading                       Lifecycle (Init/Update/Shutdown)
```

## What the engine owns

The engine is responsible for everything that produces pixels or sound:
windowing, rendering, scene management, physics, audio playback, and asset
loading. The developer chooses the engine; PlayOS does not prescribe one.

## What the Platform API owns

The Platform API is responsible for platform services that are independent
of rendering: input (in logical PlayOS terms), storage paths, capability
discovery, lifecycle, and runtime features (overlay, cloud saves,
achievements). These are the same regardless of which engine is rendering.

## Coexistence rules

1. **Separate headers** — `engine.h` and `playos/playos.h` appear in the
   same source file. Neither includes the other.
2. **Separate loops** — the engine's frame loop and `Lifecycle::Update`
   coexist in the same `while (running)` block. `Update` is called once
   per frame, before or after rendering.
3. **No type sharing** — the Platform API does not use engine types
   (`Vector2`, `Texture2D`, `Color`). The engine does not use Platform
   API types (`Button`, `Axis`, `Capability`).
4. **Input independence** — the engine may have its own input system for
   keyboard, mouse, and raw controller access. The Platform API input
   (`Button::A`, `Axis::LeftX`) is a parallel, logical layer. Both can
   coexist in the same application.
5. **Lifecycle ownership** — the application's `main()` owns both the
   engine window and `Lifecycle::Init/Shutdown`. Neither the engine nor
   the Platform API manages the other's lifecycle.

## Input bridging

For convenience, engine integration kits may provide **input bridging**
helpers that feed engine input events into PlayOS logical buttons. This
is a kit responsibility, not a Platform API responsibility. See
[Input Bridging](07-input-bridging.md).

## Requirements

- The Platform API MUST compile and link without any engine installed.
- Engine integration kits MUST NOT modify the Platform API headers.
- Every engine kit MUST demonstrate the dual-include pattern.

## Related

- [Engine-Agnostic principle](../02-platform-principles/02-engine-agnostic.md)
- [API Design Rules](../06-platform-api/02-api-design-rules.md)
- [Raylib Reference Kit](03-raylib-reference-kit.md)
