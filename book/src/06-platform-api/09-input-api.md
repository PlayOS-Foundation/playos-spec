# Input API

The `PlayOS::Input` module provides controller, keyboard, and touch input
through a single, portable API that uses **logical** buttons and axes rather
than device-specific codes.

## Logical buttons

```cpp
enum class Button {
    A, B, X, Y,
    DPadUp, DPadDown, DPadLeft, DPadRight,
    L1, R1, L2, R2,
    Start, Select,
    Home, QuickSettings,
    Count
};
```

## Logical axes

```cpp
enum class Axis {
    LeftX, LeftY,          // stick values [-1, 1]
    RightX, RightY,
    LeftTrigger,           // trigger values [0, 1]
    RightTrigger,
};
```

## API

```cpp
namespace PlayOS {
namespace Input {

void Update();                           // Poll the input backend.
bool Down(Button button);                // True while held.
bool Pressed(Button button);             // True on the first frame.
float GetAxis(Axis axis);                // Current axis value.

} // namespace Input
} // namespace PlayOS
```

## Usage

```cpp
// Continuous movement (held).
if (PlayOS::Input::Down(PlayOS::Button::DPadRight)) { MoveRight(); }

// Discrete actions (first frame only).
if (PlayOS::Input::Pressed(PlayOS::Button::A)) { Confirm(); }
if (PlayOS::Input::Pressed(PlayOS::Button::Home)) { ReturnToShell(); }

// Analog input.
float steer = PlayOS::Input::GetAxis(PlayOS::Axis::LeftX);
```

## Device mapping

Physical inputs are mapped to logical buttons and axes by the device profile
and input backend. The Ally's Armoury button → `Button::Home`; its
built-in gamepad → `Button::A/B/X/Y`. The mapping is not hard-coded in the
application. See the
[ROG Ally Reference](../12-device-model-and-porting/12-rog-ally-reference.md)
for an example device profile.

## Backends

`PlayOS::Input` delegates to an `IInputBackend` selected at build time:

| Platform | Backend | Status |
|---|---|---|
| Windows | XInput (gamepad 0) | ✅ Implemented |
| Linux | evdev (first gamepad found) | ✅ Implemented (compiles; to be validated on hardware) |
| Other | Null (reports no input) | ✅ Placeholder |

Adding a backend does not change the public API — see the
[Backend Porting Model](../12-device-model-and-porting/03-backend-porting-model.md).

## Requirements

- `Update` MUST be called once per frame (by `Lifecycle::Update`).
- `Pressed` MUST return true only on the first frame a button transitions
  from up to down.
- `Down` and `Pressed` MUST be valid for every enumerator in `Button`;
  unsupported buttons return false.
- `GetAxis` MUST return a value in the documented range; unsupported axes
  return 0.
- Backends MUST NOT require root (seat access is obtained via seatd/libseat).
