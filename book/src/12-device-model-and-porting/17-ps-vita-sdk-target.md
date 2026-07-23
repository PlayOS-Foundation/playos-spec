# PS Vita SDK Target

The **PlayStation Vita** is a historical handheld SDK Target. It
validates PlayOS portability to constrained, non-Linux hardware —
a 2011 ARM device with a custom OS and proprietary graphics API.

## Role

The PS Vita serves as a **portability proving ground**:
- If PlayOS can target the Vita, it can target anything.
- The Vita's constraints (333 MHz ARM, 512 MB RAM, no modern GPU API)
  force clean separation between the Platform API and platform
  assumptions.
- Homebrew toolchain (VITASDK, taiHEN) provides a real shipping target.

## Backend coverage

All PS Vita backends use the SCE (System Control Engine) APIs:

| Module | Planned backend | Key API |
|---|---|---|
| Input | `vita_input_backend.cpp` | `sceCtrlPeekBufferPositive`, `sceTouch` |
| Display | `vita_display_backend.cpp` | `sceDisplayGetFrameBuf` |
| Storage | `vita_storage_backend.cpp` | `ux0:data/` path |
| Audio | `vita_audio_backend.cpp` | `sceAudioOut*` |
| Battery | `vita_battery_backend.cpp` | `scePowerGetBatteryLifePercent` |
| Network | `vita_network_backend.cpp` | `sceNet` |
| Touch | `vita_touch_backend.cpp` | `sceTouchPeek` (front + rear) |

**All backends are missing.** Like Android, this is a greenfield target.

## Input mapping

Vita has physical buttons + front multitouch + rear touchpad:

```cpp
// vita_input_backend.cpp — mapping SCE_CTRL_* to PlayOS Button
switch (buttons) {
    case SCE_CTRL_CROSS:    set(Button::A, pressed); break;
    case SCE_CTRL_CIRCLE:   set(Button::B, pressed); break;
    case SCE_CTRL_SQUARE:   set(Button::X, pressed); break;
    case SCE_CTRL_TRIANGLE: set(Button::Y, pressed); break;
    case SCE_CTRL_LTRIGGER: set(Button::L1, pressed); break;
    case SCE_CTRL_RTRIGGER: set(Button::R1, pressed); break;
    case SCE_CTRL_SELECT:   set(Button::Select, pressed); break;
    case SCE_CTRL_START:    set(Button::Start, pressed); break;
    case SCE_CTRL_PSBUTTON: set(Button::Home, pressed); break;
}
// Left analog stick: SCE_CTRL_LX/LY → Axis::LeftX/LeftY
// Right analog stick: SCE_CTRL_RX/RY → Axis::RightX/RightY
```

## Display

Vita display is fixed hardware: 960×544 at ~60 Hz (OLED or LCD
depending on model). The display backend returns hardcoded values
from `sceDisplayGetFrameBuf`.

## Unique capabilities

The PS Vita has hardware capabilities not present in the core
capability registry. These would be exposed as **vendor extensions**
(see [Capability Model](../05-capability-model/01-overview.md)):

- **Rear touchpad** — `vita.touch.rear` vendor extension.
- **Cameras** — front and rear cameras (future).
- **Motion sensors** — accelerometer + gyroscope via `sceMotion`.
- **OLED vs LCD** — `sceDisplayIsOled` for model-specific behaviour.

## Build setup

```cmake
# Using VITASDK toolchain:
cmake -DCMAKE_TOOLCHAIN_FILE=$VITASDK/share/vita.toolchain.cmake \
      -DPLAYOS_TARGET=vita \
      ..
# Output: playos.suprx (kernel module) or playos.a (static lib)
```

## Limitations

- No PlayOS Shell on Vita. Games are standalone homebrew `.vpk`
  packages.
- No cloud features (PlayOS Cloud requires internet, which the Vita
  has, but the cloud API client is not ported).
- No overlay, game switching, or notifications.
- Limited to 444 MHz CPU in standard mode (333 MHz GPU). High-clock
  mode (500 MHz) is available via taiHEN plugins but not required.

## Why it matters

The PS Vita target proves that the Platform API is not coupled to:
- Linux (Vita runs a custom RTOS kernel).
- x86_64 (Vita is ARM Cortex-A9).
- Modern GPUs (Vita uses a PowerVR SGX543MP4+ with `sceGxm` API).
- Large memory (512 MB RAM, 128 MB VRAM).

This validates the engine-agnostic, platform-agnostic design
principles in [Platform Principles](../02-platform-principles/01-specification-first.md).

## Related

- [SDK Target Bringup](11-sdk-target-bringup.md)
- [Android SDK Target](16-android-sdk-target.md)
- [Backend Porting Model](03-backend-porting-model.md)
