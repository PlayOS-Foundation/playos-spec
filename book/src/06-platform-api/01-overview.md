# Overview

The PlayOS Platform API is the **engine-agnostic** contract that applications
call for platform services. It provides a stable, simple surface — input,
storage, display, lifecycle, capabilities, and more — that does not depend on
any game engine.

## The surface (as of the current vertical-slice release)

The reference implementation in
[`playos-platform-api`](https://github.com/PlayOS-Foundation/playos-platform-api)
(currently exposes three modules; more are specified in this part and
implemented incrementally):

| Module | Public header | Description |
|---|---|---|
| Lifecycle | `playos/lifecycle.h` | `Init` / `Update` / `Shutdown` — the minimal application loop |
| Input | `playos/input.h` | Logical buttons (`Button::A`, `Button::Home`) and axes (`Axis::LeftX`), `Down`/`Pressed`/`GetAxis` |
| Capabilities | `playos/capabilities.h` | `Has(capability)` / `List()` / `Id()` — runtime feature discovery (RFC-0003) |

The umbrella header `playos/playos.h` includes all three and defines the
platform version.

## Two layers, one contract

The Platform API has two layers that applications must distinguish:

- **Core (portable) subset** — available everywhere, on both
  [Runtime Devices](../04-target-model/02-runtime-devices.md) and
  [SDK Targets](../04-target-model/03-sdk-targets.md). These are the
  [Required Capabilities](../05-capability-model/03-required-capabilities.md).
- **Runtime-only features** — only on devices that run the full PlayOS
  Runtime (system overlay, package installation, suspend/resume). These are
  discovered through [Optional Capabilities](../05-capability-model/04-optional-capabilities.md).

Applications query capabilities rather than assuming which layer is present
(see [Capabilities API](08-capabilities-api.md) and RFC-0003).

## Engine-agnostic design

The Platform API must never include a game-engine header. Raylib, SDL, Godot,
a custom engine — whatever the developer chooses for rendering sits **above**
the Platform API, never inside it. See the
[Engine-Agnostic principle](../02-platform-principles/02-engine-agnostic.md)
and [API Design Rules](02-api-design-rules.md).

## Backends

Each module is implemented by a platform-specific **backend** selected at
build time by CMake. For example, `PlayOS::Input` delegates to an
`IInputBackend` that on Windows talks to **XInput**, on Linux reads **evdev**
directly, and on the PS Vita uses `sceCtrl`. The public API does not change.
Backend development is covered in the
[Device Model and Porting](../12-device-model-and-porting/01-overview.md) part.

## Versioning

The Platform API version is a `MAJOR.MINOR.PATCH` triple defined in
`playos/playos.h`. Breaking changes follow the
[API Versioning](25-api-versioning.md) policy, which preserves long-term
compatibility per the [Platform Promise](../01-vision/05-platform-promise.md).
