# Runtime

The **PlayOS Runtime** is the console operating environment on [Runtime Devices](../04-target-model/02-runtime-devices.md). It is implemented in [`playos-runtime`](https://github.com/PlayOS-Foundation/playos-runtime) and specified in [Part VIII](../08-runtime-architecture/01-overview.md).

## Responsibilities

- Launch, supervise, and clean up applications.
- Host the compositor that owns presentation and focus.
- Route input according to runtime policy.
- Provide runtime capabilities such as overlays, game switching, suspend/resume, package execution, and marketplace integration.
- Integrate portable PlayOS services with the host operating system.

## Reference implementation

| Component | Detail |
|---|---|
| Reference OS | Alpine Linux, musl, apk, and OpenRC ([ADR-0004](../../../adr/0004-use-alpine-linux-reference-os-base.md)) |
| Compositor | Dedicated C++ compositor on wlroots ([ADR-0002](../../../adr/0002-use-wlroots-tinywl-compositor.md)) |
| Process supervisor | PlayOS Runtime process groups and lifecycle state |
| Shell | Replaceable Raylib Wayland client |
| Reference device | ASUS ROG Ally |

Alpine is the base of the reference OS, not a requirement for every PlayOS implementation. Distribution-specific image, package, and init integration belongs in `playos-refdistro`.

## Where the runtime lives

The runtime library, `playos-run`, runtime services, and Linux compositor live in `playos-runtime`. Linux-specific code remains isolated behind platform build options. The runtime must not expose apk, OpenRC, musl, or wlroots as public Platform API contracts.

See the [Repository Map](12-repository-map.md) and [Linux Reference Runtime](../08-runtime-architecture/04-linux-reference-runtime.md).

## Runtime vs SDK Target

An SDK Target does not run the full reference OS or Runtime. It implements the portable Platform API, or a declared subset, through a target backend. Applications discover optional features through capabilities.

See [SDK Targets](../04-target-model/03-sdk-targets.md) and [RFC-0002](https://github.com/PlayOS-Foundation/playos-spec/blob/main/rfcs/0002-runtime-devices-and-sdk-targets.md).
