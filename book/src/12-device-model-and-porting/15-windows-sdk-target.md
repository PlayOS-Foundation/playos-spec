# Windows SDK Target

The **Windows SDK Target** enables developers to build and test
PlayOS games on Windows using the PlayOS Platform API. The API builds
as a native Windows library; games link against it and run in windowed
mode for development.

## Role

Windows is an **SDK Target** — not a primary Runtime Device.
Developers write PlayOS games on Windows, test locally, then deploy
to Linux-based Runtime Devices (ROG Ally, Steam Deck, etc.).

## Backend coverage

| Module | Backend | Status |
|---|---|---|
| Input | `windows_input_backend.cpp` | ✅ XInput for Xbox controllers |
| Display | Null | ❌ Needs Win32 impl |
| Storage | `windows_storage_backend.cpp` | ✅ `%LOCALAPPDATA%` paths |
| Audio | Null | ❌ Needs WASAPI impl |
| Battery | `windows_battery_backend.cpp` | ✅ `GetSystemPowerStatus` |
| Network | `windows_network_backend.cpp` | ⚠️ State/IP works, scan/connect stub |
| Touch | Null | ❌ Needs Windows touch injection API |
| Bluetooth | `windows_bluetooth_backend.cpp` | ✅ Basic presence check |

## Key gaps

The Windows display and audio backends are missing. Until these are
implemented:
- Display uses the null backend (returns 0×0, 0 Hz). Use Raylib's
  backend as a workaround for development.
- Audio uses the null backend (volume 1.0, not muted). Use Raylib's
  backend for sound during development.

## Build setup

```cmake
# In CMakeLists.txt:
if(CMAKE_SYSTEM_NAME STREQUAL "Windows")
    target_sources(PlayOS PRIVATE
        src/backends/windows/windows_input_backend.cpp
        src/backends/windows/windows_storage_backend.cpp
        src/backends/windows/windows_battery_backend.cpp
        src/backends/windows/windows_network_backend.cpp
        src/backends/windows/windows_bluetooth_backend.cpp
        # Use Raylib for display and audio until native backends exist
        src/backends/raylib/raylib_display_backend.cpp
        src/backends/raylib/raylib_audio_backend.cpp
    )
endif()
```

## Raylib fallback

When native Windows backends are missing, the Raylib backend is the
recommended fallback. Raylib runs on Windows via OpenGL or DirectX,
providing display info, input, and audio. This is acceptable for SDK
Target development but not for a certified Runtime Device.

## Windows as a Runtime Device

Windows is **not** a certified Runtime Device target. The PlayOS
experience on Windows is for development only. A future Windows-based
handheld (e.g., future ROG Ally running Windows) could potentially be
a Runtime Device if all backends are implemented, but this is not a
current goal.

## Related

- [SDK Target Bringup](11-sdk-target-bringup.md)
- [Android SDK Target](16-android-sdk-target.md)
- [Backend Porting Model](03-backend-porting-model.md)
- [Generic Linux PC Reference](13-generic-linux-pc-reference.md) — runtime equivalent
