# Power API

The Power API covers battery status and display brightness. These are
separate optional capabilities — a device may have a battery but no
brightness control, or vice versa.

## Battery

Gated on the optional capability `power.battery` (enum: `Capability::Battery`).

```cpp
namespace PlayOS {
namespace Battery {

float Level();        // charge level [0.0, 1.0]; returns -1.0 if absent
bool  IsCharging();   // true if connected to external power

} // namespace Battery
} // namespace PlayOS
```

### Usage

```cpp
if (PlayOS::Capabilities::Has(PlayOS::Capability::Battery)) {
    float level = PlayOS::Battery::Level();
    bool  charging = PlayOS::Battery::IsCharging();

    if (!charging && level < 0.15f) {
        ShowLowBatteryWarning();
    }
}
```

### Failure behavior

- `Level()` returns `-1.0` when the `Battery` capability is absent or the
  level cannot be determined.
- `IsCharging()` returns `false` when on battery or when the capability is
  absent.

## Brightness

Gated on the optional capability `display.brightness` (enum: `Capability::Brightness`).

```cpp
namespace PlayOS {
namespace Brightness {

float Get();          // backlight level [0.0, 1.0]; returns -1.0 if absent
void  Set(float val); // clamped to [0.0, 1.0]; no-op when absent

} // namespace Brightness
} // namespace PlayOS
```

### Usage

```cpp
if (PlayOS::Capabilities::Has(PlayOS::Capability::Brightness)) {
    PlayOS::Brightness::Set(0.7f); // 70% brightness
}
```

### Failure behavior

- `Get()` returns `-1.0` when the `Brightness` capability is absent.
- `Set()` is a no-op when the capability is absent.

## Capabilities

| Capability | Enum | Status |
|---|---|---|
| `power.battery` | `Capability::Battery` | Optional |
| `display.brightness` | `Capability::Brightness` | Optional |

## Requirements

- Battery and brightness are independent capabilities — a desktop without
  a battery can still support brightness control.
- `SetBrightness` MUST clamp input to `[0.0, 1.0]`.
- On handhelds, brightness control SHOULD affect the built-in display
  backlight.
- Battery level SHOULD update at least once per second when queried.

## Implementation note

The reference implementation (`playos/battery.h`) delegates to platform
backends: Linux reads `/sys/class/power_supply/`, Windows uses
`GetSystemPowerStatus`. The null backend returns `-1.0` / `false`.

## Related

- [Capability Failure Behaviour](../05-capability-model/06-capability-failure-behaviour.md)
- [Power Porting](../12-device-model-and-porting/07-power-porting.md)
- [ROG Ally Reference](../12-device-model-and-porting/12-rog-ally-reference.md)
