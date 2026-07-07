# Runtime

The **PlayOS Runtime** is the console operating environment that runs on
[Runtime Devices](../04-target-model/02-runtime-devices.md). It is
implemented in
[`playos-runtime`](https://github.com/PlayOS-Foundation/playos-runtime)
and specified in [Part VIII — Runtime Architecture](../08-runtime-architecture/01-overview.md).

## Responsibilities

- Own the application lifecycle: launch, supervise, and clean up games (see
  [Package Execution](../08-runtime-architecture/07-package-execution.md)).
- Host the compositor that owns the display (see
  [Compositor Model](../08-runtime-architecture/05-compositor-model.md)).
- Route input through the compositor (see
  [Input Routing](../08-runtime-architecture/10-input-routing.md)).
- Provide runtime-only capabilities: system overlay, game switching,
  suspend/resume, package installation, and the marketplace client.
- Integrate with the operating system (boot, seat management, audio,
  networking).

## Reference implementation

| Component | Detail |
|---|---|
| OS base | Arch Linux (ADR-0003) |
| Compositor | wlroots / TinyWL, written in C++ (ADR-0002, Stage 1 C skeleton in place) |
| Process launcher | `LaunchAndWait` — CreateProcess (Windows), fork/exec (POSIX) |
| Shell integration | The compositor launches the PlayOS Shell as a Wayland client |

## Where the runtime lives

The runtime library, the `playos-run` CLI, and the compositor all live in
`playos-runtime`. The compositor is Linux-only and behind a CMake option
(`-DPLAYOS_BUILD_COMPOSITOR=ON`). See the
[Repository Map](12-repository-map.md) and the
[compositor bring-up guide](https://github.com/PlayOS-Foundation/playos-runtime/blob/main/compositor/BRINGUP.md).

## Runtime vs SDK Target

On an SDK Target the runtime is not present; the Platform API provides the
portable subset only. An application discovers runtime features through
capabilities and degrades gracefully where they are absent. See
[SDK Targets](../04-target-model/03-sdk-targets.md) and
[RFC-0002](https://github.com/PlayOS-Foundation/playos-spec/blob/main/rfcs/0002-runtime-devices-and-sdk-targets.md).
