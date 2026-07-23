# Display Porting

The display backend reports the current screen dimensions, refresh
rate, and safe area insets to the Platform API.

## Interface to implement

```cpp
// backends/display_backend.h
class IDisplayBackend {
public:
    virtual DisplayInfo Query() = 0;
};
```

`DisplayInfo` contains:

```cpp
struct DisplayInfo {
    int width, height;        // Logical pixels
    int refreshRate;          // Hz
    int safeAreaLeft, safeAreaTop, safeAreaRight, safeAreaBottom; // pixels
};
```

## Linux implementation

The Linux backend uses two strategies, in priority order:

1. **Raylib `GetScreenWidth/Height`** — after the compositor has
   sized the window to the output dimensions via
   `wlr_xdg_toplevel_set_size()`, Raylib reports the correct
   compositor-assigned size. This is the authoritative source.
2. **Raylib `GetMonitorWidth/Height`** — fallback when the window
   has not yet received a configure event (returns 0×0).

Safe area insets are read from environment variables
(`PLAYOS_SAFE_AREA_LEFT`, `PLAYOS_SAFE_AREA_TOP`,
`PLAYOS_SAFE_AREA_RIGHT`, `PLAYOS_SAFE_AREA_BOTTOM`) set by the
compositor or device profile service at startup. For devices like
the ROG Ally with no physical screen obstructions (no notch,
rounded corners outside the logical grid), all insets are 0.

## Windows implementation

The Windows backend reads monitor dimensions via Win32
(`GetMonitorInfoA`, `EnumDisplaySettings`). Safe area insets are 0
for desktop monitors and most Windows devices. For devices with
notches or rounded corners (future), insets would come from the
device profile.

**Current gap: Windows display backend does not exist.** The null
backend is used. This needs implementation for a full Windows SDK
Target.

## Raylib implementation

The Raylib backend calls `GetScreenWidth/Height`,
`GetMonitorWidth/Height`, and `GetMonitorRefreshRate`. Safe areas
default to 0. This is the simplest backend and works everywhere
Raylib does.

## Safe areas

Safe area insets protect UI elements from physical screen
obstructions. For most devices (desktop monitors, ROG Ally, handheld
PCs with rectangular panels), all four values are 0. For mobile
devices with notches or rounded corners, non-zero insets are needed.

Do **not** use safe areas to create UI margins — use the layout
system for that. Safe areas are strictly for hardware obstructions.

## Porting steps

1. Create `src/backends/<platform>/<platform>_display_backend.cpp`.
2. Implement `IDisplayBackend::Query()`.
3. Call `Capabilities::RegisterCapability(Capability::DisplayInfo)`.
4. If the device has brightness control, register
   `Capability::DisplayBrightness` via a power backend (see
   [Power Porting](07-power-porting.md)).
5. Register in CMake.

## Implementation gaps

| Platform | Backend status | Notes |
|---|---|---|
| Linux | ✅ Complete | Raylib + compositor env vars |
| Raylib | ✅ Complete | Raylib API calls |
| Windows | ❌ Missing | Null backend used; needs Win32 impl |
| Android | ❌ Missing | Needs ANativeWindow query |
| PS Vita | ❌ Missing | Needs `sceDisplay` query |

## Related

- [Backend Porting Model](03-backend-porting-model.md)
- [Display Module (Capability Model)](../05-capability-model/01-overview.md)
- [Device Profile Schema](02-device-profile-schema.md)
