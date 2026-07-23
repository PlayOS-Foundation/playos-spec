# Input Porting

The input backend translates platform-specific input events into
PlayOS logical Button and Axis values.

## Interface to implement

```cpp
// backends/input_backend.h
class IInputBackend {
public:
    virtual void Update() = 0;           // Read pending events
    virtual bool Down(Button button) = 0; // Current button state
    virtual float GetAxis(Axis axis) = 0; // Current axis value [-1, 1]
    virtual bool Connected() const = 0;   // Controller physically connected
};
```

The module core (`input.cpp`) calls `Update()` once per frame and
records edge transitions so the public API can report `Pressed`,
`Released`, and `Held`.

## Linux implementation (evdev)

The Linux backend reads `/dev/input/event*` directly — no libinput
or Wayland dependency. This is deliberate: the shell must receive
controller input even while running as a Wayland client.

### Finding the gamepad

Scan `/dev/input/event*` for a device that advertises `BTN_SOUTH`
(the standard evdev code for the A button). This is a Stage-1
backend: it binds the first gamepad found. Multi-device and hotplug
support are future work.

### Standard button mapping

Standard gamepad buttons use their evdev defaults, which are
consistent across most Linux gamepad drivers:

| PlayOS Button | evdev code |
|---|---|
| `A` | `BTN_SOUTH` (304) |
| `B` | `BTN_EAST` (305) |
| `X` | `BTN_WEST` (307) |
| `Y` | `BTN_NORTH` (308) |
| `L1` | `BTN_TL` (310) |
| `R1` | `BTN_TR` (311) |
| `L2` | `BTN_TL2` (312) |
| `R2` | `BTN_TR2` (313) |
| `Select` | `BTN_SELECT` (314) |
| `Start` | `BTN_START` (315) |
| `DPad` | `BTN_DPAD_*` or `ABS_HAT0X/Y` |

### Vendor-specific buttons (Home, QuickSettings)

The Home and QuickSettings buttons are device-specific. Their evdev
codes are resolved from the [Devrice Profile](
../12-device-model-and-porting/02-device-profile-schema.md) via
`InputMapping`. This is the bridge between RFC-0006's portable input
vocabulary and Linux evdev codes:

```cpp
// input_mapping.h — resolves symbolic names → evdev codes
std::vector<int> ResolveInputCode(const std::string& symbolicName);
```

Examples:
- `"BTN_MODE"` → `{316}` (Xbox Guide button on xpad)
- `"BTN_START"` → `{315}` (Command Center on ROG Ally ASUS driver)
- `"KEY_PROG1"` → `{148}` (Armoury Crate on ASUS kernel driver)

If the profile symbol is not recognized, the backend falls back to
a hardcoded set of candidate codes.

### Axis normalization

Analog sticks are normalized to `[-1.0, 1.0]` using the device's
reported `input_absinfo` range (`ABS_X`, `ABS_Y`, `ABS_RX`, `ABS_RY`).
A 15% deadzone is applied to squash resting noise. Triggers
(`ABS_Z`, `ABS_RZ`) are normalized to `[0.0, 1.0]`.

### With and without a profile

When `PLAYOS_PROFILE` is set, the backend loads the named profile
from `/etc/playos/device-profiles/<id>.toml`. When no profile is
available, it uses hardcoded defaults suitable for any Xbox-style
controller.

## Windows implementation (XInput / Xbox)

The Windows backend uses XInput (via `windows_input_backend.cpp`).
XInput is the most reliable path for Xbox controllers on Windows.
For non-XInput controllers (DirectInput or raw HID), a future
Stage-2 backend may be needed.

## Raylib implementation

The Raylib backend delegates to `IsGamepadButtonDown()` and
`GetGamepadAxisMovement()`. Raylib's internal gamepad mapping
(SDL/GLFW-based) handles the platform abstraction. This backend is
suitable for SDK Targets on desktop OSes where Raylib is available.

## Null backend

The null input backend reports all buttons released and all axes at
0.0. `Connected()` returns false. Use it as a starting point when
porting to a platform without a controller.

## Porting steps

1. Create `src/backends/<platform>/<platform>_input_backend.cpp`.
2. Implement `IInputBackend` — at minimum, `Update()`, `Down()`, `GetAxis()`.
3. Call `Capabilities::RegisterCapability(Capability::InputBasic)`.
4. If the platform has touch, see [Display Porting](05-display-porting.md).
5. Add a device profile mapping vendor-specific buttons if needed.
6. Register in CMake.

## Implementation gaps

| Platform | Backend status | Notes |
|---|---|---|
| Linux | ✅ Complete | evdev direct input |
| Windows | ✅ Complete | XInput for Xbox controllers |
| Raylib | ✅ Complete | SDL/GLFW abstraction |
| Android | ❌ Missing | Needs Android Input API mapping |
| PS Vita | ❌ Missing | Needs `sceCtrl` mapping |

## Related

- [Device Profile Schema](02-device-profile-schema.md)
- [Backend Porting Model](03-backend-porting-model.md)
- [ROG Ally Reference](12-rog-ally-reference.md)
