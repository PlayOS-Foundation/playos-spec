# Layered Architecture

PlayOS is a **layered platform**. Each layer has a well-defined
responsibility and a clear interface to the layers above and below.

## Layer diagram

```
┌──────────────────────────────────────────────┐
│  Layer 5: Applications & Content             │
│  .gpk packages, game logic, assets           │
├──────────────────────────────────────────────┤
│  Layer 4: Shell & UX                         │
│  Home screen, overlay, settings, launcher    │
├──────────────────────────────────────────────┤
│  Layer 3: Runtime Services                   │
│  App lifecycle, IPC, cloud sync, security     │
├──────────────────────────────────────────────┤
│  Layer 2: Platform API                       │
│  C++ API: Input, Display, Audio, Storage...  │
├──────────────────────────────────────────────┤
│  Layer 1: Platform Backends                  │
│  Per-OS implementations of the API           │
├──────────────────────────────────────────────┤
│  Layer 0: Operating System                   │
│  Linux, Windows, Android, ...                │
└──────────────────────────────────────────────┘
```

## Layer responsibilities

| Layer | Responsibility | Portability |
|---|---|---|
| Applications | Game/app logic. Uses Platform API. | Portable (same .gpk everywhere) |
| Shell & UX | System UI. Users interact here. | Platform-specific UI |
| Runtime Services | Process management, IPC, cloud. | Per-Runtime-Device |
| Platform API | Public API contract. | Same headers everywhere |
| Platform Backends | OS-specific glue. | Per OS |
| Operating System | Kernel, drivers, filesystem. | Per device |

## The API boundary

The most important boundary in the diagram is between **Layer 2**
(Platform API) and **Layer 3** (Runtime Services). This is the line
between "portable application" and "platform implementation":

- Everything **above** the Platform API is portable application code.
- Everything **below** the Platform API is target-specific
  implementation.

## SDK Targets vs Runtime Devices

On **SDK Targets** (Windows, macOS, Linux desktop), Layers 3 and 4
are absent. Applications link directly to the Platform API (Layer 2),
which calls directly into Platform Backends (Layer 1).

On **Runtime Devices**, all layers are present. Applications run in
the Runtime sandbox with full lifecycle management, overlay, and cloud
services.

## Related

- [Platform Specification](03-platform-specification.md)
- [Runtime Architecture](../08-runtime-architecture/01-overview.md)
- [SDK Targets](../04-target-model/03-sdk-targets.md)
- [Runtime Devices](../04-target-model/02-runtime-devices.md)
