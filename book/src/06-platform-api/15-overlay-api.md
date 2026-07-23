# Overlay API

The Overlay API provides a system-level draw surface that renders on top of
the running application. It is how the shell draws the Home overlay, quick
settings, notifications, and other system UI without exiting the game.

This is a **runtime-only** feature — gated on the optional capability
`system.overlay` (enum: `Capability::Overlay`). SDK Targets report this
capability absent.

## Purpose

The overlay is the mechanism for:

- **Home overlay** — pressing Home opens the shell overlay on top of the
  running game.
- **Quick Settings** — volume, brightness, network, and power controls
  accessible without leaving the game.
- **Notifications** — system notifications rendered over the game surface.
- **Game switching** — the overlay can display a snapshot of the game
  while the player browses the library.

## API concepts

The overlay is a **compositor-managed surface**, not an in-application
widget. The application does not draw into the overlay — the shell does.
The Platform API exposes:

- **Overlay visibility** — the application can query whether the overlay is
  currently active.
- **Overlay lifecycle events** — the application is notified when the
  overlay appears (game should pause) and disappears (game should resume).
- **Overlay drawing** — the shell obtains a render surface from the
  compositor; the Platform API in the shell process orchestrates this.

## Lifecycle integration

When the overlay activates:
1. The compositor captures the current frame buffer.
2. The shell receives focus and renders its UI on the overlay surface.
3. The application receives an overlay-active event and SHOULD pause
   simulation, audio, and input processing.

When the overlay dismisses:
1. The shell releases the overlay surface.
2. The application receives an overlay-dismissed event and SHOULD resume.

## Capability

| Capability | Enum | Status |
|---|---|---|
| `system.overlay` | `Capability::Overlay` | Optional (runtime-only) |

## Failure behavior

When `Capability::Overlay` is absent (SDK Targets):
- Overlay lifecycle events never fire.
- The Home button press is still reported via `Input::Pressed(Button::Home)`
  — the application may use it for its own pause menu or ignore it.

## Related

- [Overlay Integration](../08-runtime-architecture/12-overlay-integration.md) — runtime architecture
- [Console UX Principles](../09-shell-and-ux/02-console-ux-principles.md)
- [Controller-First Navigation](../09-shell-and-ux/03-controller-first-navigation.md)
- RFC-0004 (Platform API Surface)
