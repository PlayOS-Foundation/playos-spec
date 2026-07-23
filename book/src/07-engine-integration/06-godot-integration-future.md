# Godot Integration Future

Godot is a popular open-source game engine. Integrating Godot with the
PlayOS Platform API is a **future** goal that presents unique challenges
and opportunities.

## Why Godot

Godot aligns with PlayOS values: open source, cross-platform, no
royalties, and a growing community. A Godot integration would let
developers use Godot's editor and workflow while targeting PlayOS
devices.

## Challenges

Godot manages its own application lifecycle, input system, and platform
abstractions. Integrating the PlayOS Platform API means:

1. **Lifecycle** — Godot calls `_ready()`, `_process()`, and
   `_notification()` on its own schedule. `Lifecycle::Update` must be
   called alongside, not instead of, Godot's frame loop.
2. **Input** — Godot has its own input map and action system.
   PlayOS logical buttons (`Button::A`, `Button::Home`) must be mapped
   to Godot actions or exposed as a GDExtension.
3. **Capabilities** — Godot's feature tags (`OS.has_feature()`) serve a
   similar purpose to PlayOS capabilities but are compile-time. Runtime
   capability discovery must be bridged.
4. **Packaging** — Godot's export system produces platform-specific
   binaries. A Godot export template for `.gpk` would be needed.

## Possible approaches

### GDExtension (recommended)

A GDExtension that wraps the Platform API and exposes it to GDScript
and C#:

```gdscript
# Godot script using PlayOS via GDExtension
if PlayOS.Capabilities.has("input.touch"):
    var touches = PlayOS.Input.touch_points()
```

### Engine module

A custom Godot build with the Platform API compiled in as an engine
module. More tightly integrated but requires a custom Godot build.

### Wrapper executable

A thin C++ executable that initializes both Godot (as a library) and
the Platform API. This is the most flexible approach but requires the
most glue code.

## Status

Godot integration is **future work**. A proof-of-concept GDExtension is
the recommended starting point.

## Related

- [Engine-Agnostic Contract](02-engine-agnostic-contract.md)
- [Custom Engine Integration](05-custom-engine-integration.md)
