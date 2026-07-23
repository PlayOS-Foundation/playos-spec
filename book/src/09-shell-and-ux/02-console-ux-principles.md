# Console UX Principles

The PlayOS Shell is designed for **console and handheld** use. These
principles guide all Shell UX decisions.

## 1. Controller-first

Every interaction must work with a gamepad. The primary input is:

- D-pad and left stick for navigation.
- A button for activation/confirm.
- B button for back/cancel.
- Home button for returning to the home screen.
- Quick Settings button for the overlay.

Touch and mouse are secondary input methods. A feature that requires
touch must have a gamepad equivalent.

## 2. TV-friendly

Text and UI elements must be readable from a typical TV viewing
distance (2-3 meters):

- Minimum text size: 24 px (logical).
- High contrast: ≥4.5:1 for normal text, ≥3:1 for large text.
- Focus indicator: clearly visible highlight on the focused element.
- No dense grids of small text.

## 3. Spatial navigation

Navigation is spatial, not tab-based. The d-pad moves focus between
adjacent elements. Focus wraps at screen edges. There is no invisible
focus trap.

## 4. Predictable layout

The Shell follows a consistent grid layout:

```
┌──────────────────────────────────────┐
│  [Status Bar: time, battery, wifi]   │
├──────────────────────────────────────┤
│                                      │
│  [Main Content Area]                 │
│  (grid of apps, settings list, etc.) │
│                                      │
├──────────────────────────────────────┤
│  [Navigation hints: A=Select B=Back] │
└──────────────────────────────────────┘
```

## 5. Minimal depth

Navigation should be shallow:

- Home → App grid (1 level).
- Home → Settings → Setting category → Setting (3 levels max).
- No deeply nested menus.

## 6. Fast transitions

- Menu transitions: ≤100 ms.
- Screen transitions: ≤200 ms.
- No splash screens or loading spinners for Shell navigation.
- Shell starts in ≤3 seconds from power-on.

## 7. Sound as feedback

Navigation sounds provide feedback:

- Focus move: subtle click.
- Activation: confirmation tone.
- Error: warning tone.
- Notification: distinct chime.

All sounds respect the system volume and can be disabled.

## 8. Consistent interaction model

The Shell establishes patterns that applications are encouraged to
follow:

- A = confirm, B = back, Start = pause/menu.
- Consistent focus highlight color and shape.
- Consistent dialog layout (title, body, actions).
- Consistent error and confirmation patterns.

## Related

- [Controller-First Navigation](03-controller-first-navigation.md)
- [Accessibility](14-accessibility.md)
- [Touch Interaction](04-touch-interaction.md)
- [Console-First Principle](../02-platform-principles/03-console-first.md)
