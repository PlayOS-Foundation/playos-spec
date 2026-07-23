# Touch Interaction

While PlayOS is **controller-first**, many Runtime Devices have
touchscreens (ROG Ally, future handhelds). The Shell must handle touch
gracefully as a secondary input method.

## How touch works

Touch input is treated as a **pointer** that can:

- Tap to activate (equivalent to A button).
- Drag to scroll (equivalent to right stick or d-pad repeat).
- Swipe for navigation gestures.

The Shell maps touch to controller actions transparently:

| Touch Action | Controller Equivalent |
|---|---|
| Tap | A button (activate) |
| Tap and hold | Long-press context menu |
| Swipe left | Left d-pad |
| Swipe right | Right d-pad |
| Swipe up | Up d-pad |
| Swipe down | Down d-pad |
| Two-finger tap | B button (back) |
| Pinch | Zoom (in applicable views) |

## Touch target size

Touch targets must be large enough for comfortable use:

- Minimum touch target: 48×48 dp (device-independent pixels).
- Minimum spacing between touch targets: 8 dp.
- Larger targets preferred (game controller-sized focus is naturally
  larger).

## Touch-only Shell views

Some Shell views are touch-optimized:

- **On-screen keyboard** — used for text input when no physical
  keyboard is connected.
- **Web views** — store descriptions, terms of service, and other web
  content can use touch scrolling.
- **Touch scrolling** — long lists (settings, library) support
  flick-to-scroll on touch.

## Gesture conflicts

Gestures must not conflict with controller input. The Shell:

- Treats touch and controller input as independent.
- Does not move focus when the screen is touched (prevents focus
  fighting).
- Does not interpret a touch as a focus change — only as a direct
  action.

## Touch in applications

Applications receive touch events through the Platform API when
`input.touch` is present:

```cpp
if (Input::TouchCount() > 0) {
    auto touch = Input::GetTouch(0);
    // touch.x, touch.y, touch.pressure, touch.phase
}
```

## Related

- [Controller-First Navigation](03-controller-first-navigation.md)
- [Console UX Principles](02-console-ux-principles.md)
- [Input API — Touch](../06-platform-api/10-input-api.md)
