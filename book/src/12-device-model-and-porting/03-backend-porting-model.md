# Backend Porting Model

Every Platform API module is backed by an abstract **backend
interface** in `playos-platform-api/src/backends/`. Porting PlayOS to
a new OS means implementing these interfaces for the target platform.

## Architecture

```
Application
    │
    ▼
Platform API (public header, e.g., playos/input.h)
    │
    ▼
Module core (e.g., input.cpp — edge detection, capability registration)
    │
    ▼
Backend interface (e.g., IInputBackend)    ← porting boundary
    │
    ▼
Platform backend (e.g., linux/linux_input_backend.cpp)
```

The backend interface is the **porting boundary**. Everything above it
is portable. Everything below it is platform-specific.

## Current backend interfaces

| Interface | Header | Methods | Null backend |
|---|---|---|---|
| `IInputBackend` | `input_backend.h` | `Update`, `Down`, `GetAxis`, `Connected` | `null_input_backend.cpp` |
| `IDisplayBackend` | `display_backend.h` | `Query` | `null_display_backend.cpp` |
| `IAudioBackend` | `audio_backend.h` | `GetMasterVolume`, `SetMasterVolume`, `IsMuted`, `SetMuted` | `null_audio_backend.cpp` |
| `IBatteryBackend` | `battery_backend.h` | `Level`, `IsCharging` | `null_battery_backend.cpp` |
| `INetworkBackend` | `network_backend.h` | `GetWiFiState`, `PrimaryIP`, `ScanNetworks`, `Connect` | `null_network_backend.cpp` |
| `IStorageBackend` | `storage_backend.h` | `SavePath`, `ConfigPath` | `null_storage_backend.cpp` |
| `ITouchBackend` | `touch_backend.h` | `PointCount`, `GetPoint`, `Available` | `null_touch_backend.cpp` |
| `IBluetoothBackend` | `bluetooth_backend.h` | `IsPresent` | `null_bluetooth_backend.cpp` |

## Backend factory pattern

Each backend header declares a **factory function**:

```cpp
// In input_backend.h
IInputBackend* CreateInputBackend();
```

The CMake build system selects exactly one translation unit that
implements this function per target platform:

```cmake
if(CMAKE_SYSTEM_NAME STREQUAL "Linux")
    target_sources(PlayOS PRIVATE src/backends/linux/linux_input_backend.cpp)
elseif(CMAKE_SYSTEM_NAME STREQUAL "Windows")
    target_sources(PlayOS PRIVATE src/backends/windows/windows_input_backend.cpp)
else()
    target_sources(PlayOS PRIVATE src/backends/null/null_input_backend.cpp)
endif()
```

The module core (`input.cpp`) calls `CreateInputBackend()` once at
startup and holds the pointer. No runtime backend switching.

## Capability registration

Backends register their capabilities on construction:

```cpp
// In linux_input_backend.cpp constructor
Capabilities::RegisterCapability(Capability::InputBasic);

// In linux_touch_backend.cpp constructor (if available)
Capabilities::RegisterCapability(Capability::Touch);
```

The null backend registers no optional capabilities (only required
ones, which are always present). A real backend registers all
capabilities it provides. The module core queries the backend to
decide what to register (e.g., `ITouchBackend::Available()`).

## Porting checklist

When adding a new platform `<platform>`, create:

1. `src/backends/<platform>/` directory.
2. One backend `.cpp` per interface you implement.
3. The factory function implementation per backend.
4. CMake entries selecting your backends for `<platform>`.
5. Capability registration in each backend constructor.

## Testing a backend

1. Build the Platform API with `-DPLAYOS_BACKEND=<platform>`.
2. Run the API conformance suite (`playos-cli test --suite conformance`).
3. Verify all capabilities the backend registers are correctly
   reported by `Capabilities::Has()`.
4. Verify input, display, and other modules report correct data.

## Related

- [Platform Backends](../04-target-model/06-platform-backends.md)
- Subsystem-specific chapters: [Input](04-input-porting.md), [Display](05-display-porting.md),
  [Audio](06-audio-porting.md), [Power](07-power-porting.md),
  [Storage](08-storage-porting.md), [Network](09-network-porting.md)
- [Null Backend Strategy](../04-target-model/06-platform-backends.md#null-backend)
