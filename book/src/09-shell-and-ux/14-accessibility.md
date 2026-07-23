# Accessibility

PlayOS provides **accessibility features** to ensure the platform is
usable by the widest possible audience. Accessibility is a certification
requirement for PlayOS Certified Hardware.

## Features

| Feature | Description |
|---|---|
| **Screen reader** | Reads UI elements aloud. Supports gamepad navigation. |
| **High contrast mode** | Increases contrast to ≥7:1. Inverts or simplifies colors. |
| **Large text mode** | Scales all Shell text by 1.5×. Minimum 36 px. |
| **Button remapping** | Reassign any button to any function. System-wide and per-app. |
| **Stick dead zone adjustment** | Customize left and right stick dead zones. |
| **Vibration control** | Reduce or disable haptic feedback. |
| **Subtitle support** | System-wide subtitle preferences (size, color, background). |
| **Color blind modes** | Protanopia, deuteranopia, tritanopia filters. |
| **Reduce motion** | Disable animations and transitions. |
| **Mono audio** | Combine stereo channels into mono. |

## Screen reader

The screen reader is controller-operated:

- **D-pad**: Move focus. Each element is read aloud.
- **Y**: Read current element description again.
- **X**: Read full screen summary.
- **View/Back**: Open accessibility quick menu.

The screen reader reads:
- Element title and type ("Settings button").
- Element state ("Wi-Fi toggle, on").
- Context hints ("Press A to open").

## Button remapping

Button remapping is system-wide and per-application:

```
┌──────────────────────────────────────────┐
│  ← Button Remapping                      │
├──────────────────────────────────────────┤
│  Global Profile                          │
│  A Button      [A Button       ▾]        │
│  B Button      [B Button       ▾]        │
│  X Button      [X Button       ▾]        │
│  Y Button      [Y Button       ▾]        │
│  ...                                     │
│                                          │
│  [+ Add per-app profile...]              │
└──────────────────────────────────────────┘
```

Per-application profiles override the global profile when that
application is running.

## Subtitles

The Shell provides system-wide subtitle preferences that applications
can query through the Platform API:

```cpp
Accessibility::SubtitlePrefs prefs = Accessibility::GetSubtitlePrefs();
// prefs.enabled, prefs.fontSize, prefs.fontColor, prefs.backgroundColor
```

Applications should respect these preferences in their own subtitle
rendering.

## Certification requirements

For PlayOS Certified Hardware:

| Requirement | Level |
|---|---|
| Screen reader | Required |
| Button remapping | Required |
| High contrast | Required |
| Large text | Required |
| Subtitle support | Required |
| Color blind modes | Recommended |

## Related

- [Console UX Principles](02-console-ux-principles.md)
- [Settings](08-settings.md)
- [Certification](../14-certification/01-overview.md)
- [Controller-First Navigation](03-controller-first-navigation.md)
