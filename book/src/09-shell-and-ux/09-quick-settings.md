# Quick Settings

The **Quick Settings** panel is an overlay that provides access to
frequently used settings without leaving the current application.

## Layout

```
┌──────────────────────────────────────────┐
│  Quick Settings                     [✕]  │
├──────────────────────────────────────────┤
│                                          │
│  [🔊 Volume      ████████░░  70%]        │
│  [☀  Brightness  ██████░░░░  60%]        │
│                                          │
│  [📶 Wi-Fi        Connected  ▸]          │
│  [🔵 Bluetooth   2 devices   ▸]          │
│  [🔋 Battery      85% - 3h remaining]    │
│                                          │
│  [✈  Flight Mode              ○]         │
│  [🌙 Night Mode               ●]         │
│                                          │
│  [⚙  All Settings...]                    │
│                                          │
├──────────────────────────────────────────┤
│  [Notifications area]                    │
│  No new notifications                    │
└──────────────────────────────────────────┘
```

## Controls

- **D-pad / Left stick**: Move focus.
- **Left stick L/R on slider**: Adjust slider value.
- **A**: Toggle or activate.
- **B**: Close quick settings.
- **Quick Settings button**: Toggle open/close.

## Opening and closing

The Quick Settings panel is opened by pressing the Quick Settings
button (mapped in the device profile). It slides in from the right
edge of the screen. It closes when:

- The Quick Settings button is pressed again.
- B is pressed.
- An option navigates to full Settings.
- The Home button is pressed.

## Interaction with applications

When Quick Settings is open:

- The application continues running but its audio is ducked.
- Input is routed to Quick Settings, not the application.
- The application can detect Quick Settings being open via
  `Overlay::IsVisible()` and pause gameplay.

## Performance HUD (Developer Mode)

In developer mode, the Quick Settings panel includes a performance HUD:

```
┌──────────────────────────────────────────┐
│  Performance                             │
│  FPS: 59.8   Frame: 16.2ms              │
│  CPU: 34%    GPU: 62%    RAM: 1.2GB     │
│  Drops: 2    Temp: 62°C                  │
└──────────────────────────────────────────┘
```

This is never shown in production (certified) builds.

## Related

- [Overlay Integration](../08-runtime-architecture/12-overlay-integration.md)
- [Settings](08-settings.md)
- [Developer Mode](13-developer-mode.md)
- [Overlay API](../06-platform-api/15-overlay-api.md)
