# Power Porting

The power subsystem covers battery state, charging status,
brightness, and (future) TDP/performance controls.

## Interface to implement

```cpp
// backends/battery_backend.h
class IBatteryBackend {
public:
    virtual float Level() = 0;        // Battery level [0.0, 1.0], or -1.0
    virtual bool IsCharging() = 0;    // Connected to external power
};
```

Brightness and TDP control are not part of `IBatteryBackend` in the
current Platform API. They are expected to be added in a future
`IPowerBackend` or as extensions to `IBatteryBackend`.

## Linux implementation (sysfs)

The Linux battery backend reads `/sys/class/power_supply/`. This
works on any Linux device that exposes a standard power supply node:
handheld PCs, laptops, single-board computers with battery hats.

### Finding the battery

Scan `/sys/class/power_supply/` for a directory whose `type` file
contains `"Battery"`. The backend caches the result so subsequent
calls don't re-scan.

### Reading level

The backend prefers `energy_full` / `energy_now` (in µWh) for
higher precision, falling back to `capacity` (percentage × 100).
Returns -1.0 if no battery is present or the node is unreadable.

### Reading charging state

The `status` sysfs file can be:
- `"Charging"` — connected to external power
- `"Discharging"` — on battery
- `"Full"` — battery full, external power connected
- `"Not charging"` — connected but not charging (e.g., battery health management)

The backend returns `true` for both `"Charging"` and `"Full"`.

## Windows implementation

The Windows battery backend uses `GetSystemPowerStatus` from the
Win32 API. Level is read from `BatteryLifePercent`, charging state
from `ACLineStatus`. This covers all Windows laptops, tablets, and
handheld PCs.

## Brightness control

Brightness is a separate concern from battery. On Linux, read/write
`/sys/class/backlight/` (sysfs). On Windows, use
`GetMonitorBrightness` / `SetMonitorBrightness` (Win32 monitor API).
Brightness is registered as `Capability::DisplayBrightness`.

**Current gap: Brightness backends do not exist on any platform.**
The battery backend does not control display brightness. Both Linux
and Windows need a brightness implementation.

## TDP and performance controls

Handheld gaming PCs (ROG Ally, Steam Deck) expose vendor-specific
TDP/power profiles. These are device-specific vendor extensions, not
part of the core Platform API. Future work may define a
`Capability::PowerTDP` with a `IPowerBackend` interface.

## Porting steps

1. Create `src/backends/<platform>/<platform>_battery_backend.cpp`.
2. Implement `IBatteryBackend`.
3. Call `Capabilities::RegisterCapability(Capability::Battery)` if a
   battery is present.
4. For brightness, implement a separate backend or extend the battery backend.
5. Register in CMake.

## Implementation gaps

| Platform | Battery | Brightness | Notes |
|---|---|---|---|
| Linux | ✅ Complete (sysfs) | ❌ Missing | sysfs backlight |
| Windows | ✅ Complete (Win32) | ❌ Missing | Win32 monitor API |
| Raylib | ❌ Missing | ❌ Missing | Null backend used |
| Android | ❌ Missing | ❌ Missing | Need `BatteryManager` |
| PS Vita | ❌ Missing | ✅ Via `sceDisplay` | Battery via `scePower` |

## Related

- [Backend Porting Model](03-backend-porting-model.md)
- [Display Porting](05-display-porting.md) — brightness control
