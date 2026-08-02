# ADR-0006 — Raylib for Shell and Game UI

**Date:** Sprint 5  
**Status:** Accepted  
**Deciders:** PlayOS core team

---

## Context

`playos-shell`, `playos-overlay`, and games need a rendering framework. Options: Raylib, SDL2+OpenGL, raw OpenGL ES, Qt, GTK, custom engine.

## Decision

Use Raylib as the rendering framework for the shell, overlay, and recommended game development. Implement a custom Raylib PlayOS backend (`rcore_playos.c`) that integrates with the Wayland/EGL surface and the `libplayos` lifecycle API.

## Rationale

- **Game-developer-friendly:** Raylib is a beginner-to-intermediate game framework with a clean C API — matches the target game developer audience
- **Wayland support:** Raylib already has Wayland/EGL support — the PlayOS backend extends this rather than starting from scratch
- **Minimal dependencies:** Raylib has very few external dependencies; works well in a musl/Buildroot environment
- **Console-appropriate:** Raylib is designed for fullscreen games — no window management, no decorations, no desktop assumptions
- **C ABI compatible:** Raylib's C API is compatible with the `libplayos` C ABI philosophy
- **Active community:** Regular upstream releases; ROG Ally and AMDGPU-based hardware known to work with Raylib/Wayland

## Wayland Backend Approach

Rather than using Raylib's generic Wayland backend as-is, PlayOS implements `rcore_playos.c` which:
- Creates a fullscreen `xdg_toplevel` with trusted role environment variables
- Integrates `playos_lifecycle_poll()` into the Raylib frame loop
- Maps `playos_input_get_controller_state()` to Raylib input events
- Disables desktop features (resize, decorations, clipboard, multi-window)

## Alternatives Considered

| Option | Rejected because |
|---|---|
| SDL2 | Heavier; desktop-oriented features; PlayOS would need to strip a lot |
| Raw OpenGL ES | No windowing abstraction — more code to write for the shell UI |
| Qt | Very heavy; complex build; LGPL licensing concerns for static linking |
| GTK | Desktop-oriented; heavy; Wayland support has desktop assumptions |
| Godot | Full game engine is overkill for the shell; heavy binary size |

## Consequences

- Games targeting PlayOS are recommended to use Raylib, but the `libplayos` C ABI is engine-agnostic — SDL2 or other frameworks can be adapted
- Raylib version must be pinned in `versions.lock`
- The `rcore_playos.c` backend must be maintained when Raylib updates change platform backend APIs
- Multi-surface support (shell + overlay as one process) is limited by Raylib's single-surface design — the overlay is a separate process as a result (see [ADR for overlay architecture])
