# Rendering Boundaries

The rendering boundary defines what the engine owns and what the Platform
API provides for display. The split is simple: the engine renders; the
Platform API informs.

## What the engine owns

- **Window creation** — the engine creates and manages the application
  window (or fullscreen surface).
- **Rendering context** — OpenGL, Vulkan, DirectX, Metal — the engine
  manages the graphics API.
- **Frame buffer** — the engine owns the frame buffer and swap chain.
- **Viewport and camera** — projection, view matrix, FOV, near/far planes.

## What the Platform API provides

- **Display properties** — `Display::Current()` returns width, height,
  refresh rate, and safe area insets. The engine uses these to configure
  its window and viewport.
- **Safe area** — insets for notches, rounded corners, and bezels. The
  engine SHOULD avoid rendering critical UI in the unsafe area.
- **Overlay surface** — the compositor-managed overlay surface is above
  the engine's frame buffer. The engine does not draw into it; the
  shell does.

## The overlay boundary

```text
┌─────────────────────────────────┐
│  Shell overlay (topmost)        │  ← compositor-managed
├─────────────────────────────────┤
│  Engine frame buffer            │  ← application rendering
└─────────────────────────────────┘
```

When the overlay is active, the engine's frame buffer is captured and
displayed behind the shell UI. The engine itself is paused.

## Resolution independence

The Platform API does not mandate a specific resolution. The engine
chooses its render target; `Display::Current()` provides the physical
display properties. The engine may render at a different resolution
and scale.

## Requirements

- The Platform API MUST NOT create a window, rendering context, or
  frame buffer.
- `Display::Current()` MUST return the physical display properties of
  the device, not the engine's render target.
- The engine MUST respect the safe area for critical UI elements.

## Related

- [Display API](../06-platform-api/11-display-api.md)
- [Overlay API](../06-platform-api/15-overlay-api.md)
- [Engine-Agnostic Contract](02-engine-agnostic-contract.md)
