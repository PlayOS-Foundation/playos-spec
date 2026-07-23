# Android SDK Target

The **Android SDK Target** enables PlayOS games to be built for
Android devices via the NDK. It is an SDK Target — development and
testing happen on Android hardware, with deployment to Linux Runtime
Devices as the production path.

## Role

Android is a natural SDK Target because:
- Many handheld gaming devices run Android (Retroid, AYN Odin, Aya Neo Pocket).
- The NDK toolchain is mature and well-supported.
- Android's input and display APIs map cleanly to PlayOS concepts.

## Backend coverage

| Module | Backend | Status |
|---|---|---|
| Input | `android_input_backend.cpp` | ❌ Missing |
| Display | `android_display_backend.cpp` | ❌ Missing |
| Storage | `android_storage_backend.cpp` | ❌ Missing |
| Audio | `android_audio_backend.cpp` | ❌ Missing |
| Battery | `android_battery_backend.cpp` | ❌ Missing |
| Network | `android_network_backend.cpp` | ❌ Missing |
| Touch | `android_touch_backend.cpp` | ❌ Missing |

**All Android backends are missing.** This is a greenfield porting
target.

## Input mapping

Android input uses `InputEvent` and `MotionEvent` from the NDK. A
`android_input_backend.cpp` would:

1. Register as an `InputQueue` callback in `android_main()`.
2. Map `AMOTION_EVENT_ACTION_DOWN/UP` and `AXIS_*` to PlayOS Buttons and Axes.
3. Support gamepad via `InputDevice.getSources()` check for `SOURCE_GAMEPAD`.
4. Support touch via `AMOTION_EVENT_ACTION_POINTER_DOWN/UP`.

## Display

Use `ANativeWindow_getWidth/Height()` and `AConfiguration_getScreenSize()`.
For refresh rate, query `Display.getMode()` via JNI or use the NDK
`ADisplay` API (API 30+).

## Storage

Use `Context.getFilesDir()` via JNI for save/config paths:

```cpp
// In android_storage_backend.cpp:
// SavePath = <internal_storage>/files/PlayOS/saves/
// ConfigPath = <internal_storage>/files/PlayOS/config/
```

## Build setup

```cmake
# CMake toolchain provided by NDK:
cmake -DCMAKE_TOOLCHAIN_FILE=$NDK/build/cmake/android.toolchain.cmake \
      -DANDROID_ABI=arm64-v8a \
      -DANDROID_PLATFORM=android-26 \
      -DPLAYOS_TARGET=android \
      ..
```

## Limitations

- The PlayOS Shell does not run on Android natively. Games built for
  Android are tested as standalone `.apk` packages.
- Cloud features (cloud saves, achievements, leaderboards) require
  network connectivity and PlayOS account login.
- No overlay, game switching, or system notifications on Android.

## Alternatives: Android as Runtime Device?

Some Android-based handhelds (Retroid Pocket, AYN Odin) could
potentially be Runtime Devices if the PlayOS compositor and shell are
ported to Android's graphics stack (SurfaceFlinger/EGL). This is
future work and not a current goal.

## Related

- [SDK Target Bringup](11-sdk-target-bringup.md)
- [Windows SDK Target](15-windows-sdk-target.md)
- [Backend Porting Model](03-backend-porting-model.md)
