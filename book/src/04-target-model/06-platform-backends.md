# Platform Backends

A **platform backend** is the OS-specific implementation of a Platform API
module. Backends are what make the same API work on Linux, Windows, Android,
and other targets.

## Backend model

Each Platform API module defines an abstract backend interface in
`src/backends/`. Platform-specific implementations live in subdirectories:

```text
src/backends/
├── input_backend.h            # Abstract interface
├── linux/
│   ├── linux_input_backend.cpp  # Linux implementation (evdev)
│   └── input_mapping.cpp        # Device-profile key mapping
├── windows/
│   └── windows_input_backend.cpp  # Windows implementation (XInput)
├── null/
│   └── null_input_backend.cpp     # Placeholder (reports no input)
└── raylib/
    └── raylib_input_backend.cpp   # Raylib-based (used by shell)
```

## Backend selection

The build system (CMake) selects backends based on the target platform:

```cmake
if(CMAKE_SYSTEM_NAME STREQUAL "Linux")
    target_sources(PlayOS PRIVATE src/backends/linux/linux_input_backend.cpp)
elseif(CMAKE_SYSTEM_NAME STREQUAL "Windows")
    target_sources(PlayOS PRIVATE src/backends/windows/windows_input_backend.cpp)
else()
    target_sources(PlayOS PRIVATE src/backends/null/null_input_backend.cpp)
endif()
```

The public API does not change. `Input::Pressed(Button::A)` calls into
whatever backend was compiled in.

## Null backend

Every module has a **null backend** — a conformant implementation that
reports the capability absent and returns safe defaults. The null backend
is the fallback for unsupported platforms and the starting point for
porting to a new platform.

## Backend interface stability

Backend interfaces are internal to the Platform API implementation. They
may change between MINOR versions without affecting the public API. Only
the public headers in `include/playos/` are covered by the compatibility
guarantee.

## Related

- [Backend Porting Model](../12-device-model-and-porting/03-backend-porting-model.md)
- [Input Porting](../12-device-model-and-porting/04-input-porting.md)
- [API Design Rules](../06-platform-api/02-api-design-rules.md)
