# Console First

PlayOS is designed **console-first**: the primary experience is on a
television or handheld, navigated with a controller, and optimized for
a lean-back, immersive session.

## What "console-first" means

| Aspect | Console-first approach |
|---|---|
| **Input** | Controller is the primary input device. Every part of the platform (shell, settings, store) is navigable with a D-pad and face buttons. |
| **Display** | Designed for a TV or handheld screen at a distance. Large text, high contrast, safe areas for overscan and notches. |
| **Interaction** | Lean-back, minimal typing. On-screen keyboard as a fallback, not the default. |
| **Lifecycle** | Suspend and resume. Quick game switching. No "close the window" concept. |
| **Audio** | System-wide volume, not per-window. Mute is a physical expectation. |
| **Performance** | Consistent frame rate over peak throughput. No background tasks that steal frame budget. |

## Not exclusive

"Console-first" does not mean "console-only." The Platform API works on
desktop SDK Targets (Windows, macOS, Linux) where keyboard and mouse are
the primary input. The principle means that the console experience is the
design target; desktop and mobile are accommodated but not the primary
focus.

## Controller-first navigation

Every UI in the PlayOS shell — Home screen, library, store, settings,
notifications — must be fully navigable with a controller. Touch and
mouse are secondary input methods that enhance but do not replace
controller navigation. See
[Controller-First Navigation](../09-shell-and-ux/03-controller-first-navigation.md).

## Suspend and resume

On Runtime Devices, applications suspend when the player presses Home
and resume when they return. The application's state is preserved; no
save/load cycle is required. This is a console expectation — the player
should be able to put the device to sleep mid-game and resume exactly
where they left off.

## Related

- [Controller-First Navigation](../09-shell-and-ux/03-controller-first-navigation.md)
- [Console UX Principles](../09-shell-and-ux/02-console-ux-principles.md)
- [Suspend and Resume](../08-runtime-architecture/16-suspend-and-resume.md)
