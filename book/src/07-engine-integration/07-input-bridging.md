# Input Bridging

Input bridging connects an engine's input system to the PlayOS Platform
API's logical button and axis model. It ensures `Button::Home` and
`Button::QuickSettings` work consistently regardless of the engine.

## What input bridging does

An input bridge translates engine input events into Platform API input
state:

```text
Engine input event (SDL_GamepadButton, raylib key press, Godot InputEvent)
    ↓
Input bridge (engine-specific translation)
    ↓
PlayOS logical button/axis (Button::A, Axis::LeftX)
    ↓
Application reads Input::Pressed / Input::Down / Input::GetAxis
```

## Two approaches

### 1. Backend-level bridging (recommended for kits)

The Platform API input backend directly reads the engine's input state:

```cpp
// In a Raylib input backend:
bool RaylibInputBackend::IsButtonDown(PlayOS::Button button) {
    switch (button) {
        case PlayOS::Button::A:
            return IsGamepadButtonPressed(0, GAMEPAD_BUTTON_RIGHT_FACE_DOWN);
        // ...
    }
}
```

This is how the Raylib backend in `playos-platform-api` works. The
application calls `Input::Pressed(Button::A)` and gets the correct
result regardless of the underlying engine.

### 2. Application-level bridging

The application reads engine input and manually feeds it to the Platform
API:

```cpp
// Not recommended — use backend-level bridging instead.
if (IsKeyPressed(KEY_ENTER)) {
    // manually translate to PlayOS button
}
```

This is error-prone and breaks the `Pressed`/`Down` semantics.

## Home and QuickSettings

The `Home` and `QuickSettings` buttons are special: they are handled by
the runtime (overlay activation, game switching), not by the application.
Input bridging MUST ensure these buttons are always mapped, even if the
engine does not natively recognize them.

On the ROG Ally, for example, the Armoury Crate button is mapped to
`Button::Home` and the Command Center button to `Button::QuickSettings`
via the device profile. The input bridge reads the raw evdev events and
translates them.

## Requirements

- Input bridging MUST NOT change the public Platform API.
- Every backend MUST map at least the required buttons (`A`, `B`,
  `DPadUp/Down/Left/Right`, `Home`).
- Backend-level bridging is preferred over application-level bridging.
- The device profile defines the physical-to-logical mapping; the
  input bridge implements it.

## Related

- [Input API](../06-platform-api/09-input-api.md)
- [Device Profile Schema](../12-device-model-and-porting/02-device-profile-schema.md)
- [Input Porting](../12-device-model-and-porting/04-input-porting.md)
