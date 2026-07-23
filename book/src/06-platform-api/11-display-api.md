# Display API

The `PlayOS::Display` module provides display properties — size, refresh
rate, and safe area insets. It is gated by the required capability
`display.info`.

## API

```cpp
namespace PlayOS {

struct DisplayInfo {
    int width;              // current display width in pixels
    int height;             // current display height in pixels
    int refreshRate;        // current refresh rate in Hz

    // Safe area insets in pixels (for notches, rounded corners, bezels).
    // On devices without obstructions these are all zero.
    int safeAreaLeft;
    int safeAreaTop;
    int safeAreaRight;
    int safeAreaBottom;
};

namespace Display {
    DisplayInfo Current();  // current display properties
}

} // namespace PlayOS
```

## Usage

```cpp
auto info = PlayOS::Display::Current();

// Use the safe area to avoid rendering into notches or rounded corners.
int renderX = info.safeAreaLeft;
int renderW = info.width - info.safeAreaLeft - info.safeAreaRight;

// Match the display refresh rate for smooth presentation.
int targetFPS = info.refreshRate;
```

## Safe area

The safe area represents the rectangle of the display guaranteed to be
visible and unobstructed. On devices with notches, rounded corners, or
bezels, the insets are non-zero. On standard rectangular displays, all
insets are zero.

Applications SHOULD place critical UI (health bars, menus, text) within the
safe area. Full-screen rendering (game worlds, backgrounds) MAY extend to
the full display dimensions.

## Capability

| Capability | Status | Required |
|---|---|---|
| `display.info` | Required | Yes |

## Requirements

- `Current()` MUST return the actual display dimensions of the primary
  monitor at call time.
- `refreshRate` MUST reflect the current display mode, not a fixed
  compile-time value.
- Safe area insets MUST be zero when the display has no obstructions.
- On multi-monitor systems, `Current()` reports the primary monitor.

## Implementation note

The reference implementation (`playos/display.h`) delegates to a
platform-specific `IDisplayBackend`. The Linux backend queries the active
mode via DRM/KMS or falls back to the primary X11/Wayland output.

## Related

- [Input API](09-input-api.md)
- [Device Profile Schema](../12-device-model-and-porting/02-device-profile-schema.md)
