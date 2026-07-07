# Platform API

The **PlayOS Platform API** is the engine-agnostic contract that applications
call for platform services. It is implemented in
[`playos-platform-api`](https://github.com/PlayOS-Foundation/playos-platform-api)
and specified in [Part VI — Platform API](../06-platform-api/01-overview.md).

## Role

- Provide a stable, small, portable surface for input, storage, lifecycle,
  capabilities, and other platform services.
- Be engine-agnostic: usable with Raylib, SDL, Godot, and custom engines.
- Hide platform differences behind swappable backends selected at build time
  by CMake.

## Current surface

The reference implementation currently provides three modules (more are
specified and added incrementally):

| Module | Purpose |
|---|---|
| `Lifecycle` | `Init` / `Update` / `Shutdown` — the minimal application loop |
| `Input` | Logical buttons and axes polled once per frame |
| `Capabilities` | Runtime feature discovery (`Has` / `List` / `Id`) |

## Architecture

```text
Application
    └── PlayOS::Input / Lifecycle / Capabilities   (public API)
            └── IInputBackend                      (internal interface)
                    ├── Windows: XInput
                    ├── Linux:   evdev
                    ├── (future) PS Vita: sceCtrl
                    └── Other:   null backend
```

The public API is the same on every platform. The backend is a build-time
choice. Adding a platform means adding a backend, not changing the API.

## Repository

[`playos-platform-api`](https://github.com/PlayOS-Foundation/playos-platform-api)
is the reference implementation. It is consumed by the shell, samples, and any
game that uses the Platform API. See the [Repository Map](12-repository-map.md).
