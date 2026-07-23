# SDK Target Bringup

An **SDK Target** is a platform where the PlayOS SDK builds and
links, but the full runtime shell may not run natively. SDK Targets
are for development, testing, and cross-compilation — the developer
writes code against the PlayOS Platform API and tests on a
development machine before deploying to a Runtime Device.

## What makes an SDK Target different

| Aspect | Runtime Device | SDK Target |
|---|---|---|
| Shell runs natively | Yes | Optional |
| All backends required | Yes | Only what the developer needs |
| Device profile required | Yes | Recommended |
| Conformance required | Full suite | Subset |
| Input | Native controller | Keyboard/mouse or gamepad |
| Certification | Available | Not applicable |

## Phase 1: Minimal backend set

Start with these backends (most platforms):

1. **Input** — keyboard/mouse or gamepad mapping. On Raylib targets,
   use the Raylib backend. On native targets, map the platform's
   input system.
2. **Display** — report the host monitor dimensions.
3. **Storage** — provide host-filesystem paths for save/config.
4. **Audio** — volume control (Raylib backend or native).

Optional backends for richer development:
- **Network** — if testing network features.
- **Battery** — if developing battery-aware UI.

## Phase 2: Build configuration

```cmake
# In CMakeLists.txt, add a target for the SDK platform:
if(CMAKE_SYSTEM_NAME STREQUAL "Android")
    target_sources(PlayOS PRIVATE
        src/backends/android/android_input_backend.cpp
        src/backends/android/android_display_backend.cpp
        src/backends/null/null_audio_backend.cpp
    )
endif()
```

Use **null backends** for modules that don't apply to the SDK Target.
Every module must have exactly one backend selected per build.

## Phase 3: Conformance (subset)

Run only the tests relevant to the SDK Target's capabilities:

```sh
playos-cli test --suite conformance --capabilities input,display,storage
```

Full conformance is not required unless the SDK Target is also being
certified as a Runtime Device.

## Phase 4: Cross-compilation setup

If the SDK Target's CPU architecture differs from the development
machine:

1. [ ] Install the cross-compilation toolchain (e.g., NDK for Android, VITASDK for PS Vita).
2. [ ] Create a CMake toolchain file.
3. [ ] Add a build preset: `cmake --preset android-arm64`.
4. [ ] Verify the library builds: `cmake --build build/android-arm64`.

## Platform-specific notes

| Platform | Key dependencies | See chapter |
|---|---|---|
| Windows | Visual Studio, Win32 SDK | [Windows SDK Target](15-windows-sdk-target.md) |
| Android | NDK, Android API 26+ | [Android SDK Target](16-android-sdk-target.md) |
| PS Vita | VITASDK, taiHEN | [PS Vita SDK Target](17-ps-vita-sdk-target.md) |

## Related

- [Runtime Device Bringup](10-runtime-device-bringup.md)
- [Backend Porting Model](03-backend-porting-model.md)
- [Target Model](../04-target-model/01-overview.md)
