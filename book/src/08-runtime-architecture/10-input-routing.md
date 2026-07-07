# Input Routing

Input routing decides which surface receives controller, touch, keyboard,
and mouse events. The compositor owns routing; the vertical slice needs it
so the shell and the game each receive input when in the foreground.

## Model

```text
libinput (evdev devices)
   ↓
PlayOS Compositor
   ↓  routes to the focused surface
Foreground client (Shell or Game)
```

- The compositor reads input through **libinput**.
- Input is delivered to the **foreground** surface only.
- Certain **system inputs** are intercepted by the compositor before the
  foreground client sees them.

## System inputs

The compositor reserves a small set of system gestures/buttons, most
importantly:

- **Home** — returns to the shell / opens the system overlay. The compositor
  intercepts Home regardless of which client is focused, so the player can
  always escape a running game.

Vendor buttons are mapped to logical PlayOS inputs by the device profile
(for example, the Ally's Armoury button → Home). See
[ROG Ally Reference](../12-device-model-and-porting/12-rog-ally-reference.md).

## Requirements

- The compositor MUST deliver input only to the foreground surface, except
  for reserved system inputs.
- The compositor MUST intercept **Home** globally so the shell is always
  reachable from a running game.
- Input device access MUST be obtained via seatd/libseat, not by requiring
  root.

## Vertical slice note

Minimal implementation: the compositor forwards controller input to whichever
surface is foreground (shell or game) and intercepts one reserved button
(Home / Ally Armoury) to trigger the return-to-shell path.

See: [Compositor Model](05-compositor-model.md),
[Game Switching](11-game-switching.md),
[Controller-First Navigation](../09-shell-and-ux/03-controller-first-navigation.md).
