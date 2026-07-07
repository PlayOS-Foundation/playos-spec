# Overview

This chapter gives a high-level map of the PlayOS platform architecture.
Later chapters specify each layer in detail.

## Layered architecture

PlayOS is organized into layers, each with a single responsibility:

```text
Applications / Games
        ↓
Engine (Raylib / SDL / Godot / Custom)
        ↓
PlayOS Platform API        ← the engine-agnostic contract
        ↓
Backend Implementation     ← per-target implementation
        ↓
PlayOS Runtime             ← console operating environment (runtime devices)
        ↓
Operating System + Services (Linux / wlroots / systemd / PipeWire / seatd)
        ↓
Hardware
```

Rendering is not a platform service. Platform services are not operating
system services. Operating systems are not hardware. Keeping these
boundaries clean is what makes PlayOS portable.

## Major components

| Component | Responsibility | Repository |
|---|---|---|
| **Specification** | Defines the platform contract | `playos-spec` |
| **Platform API** | Engine-agnostic services | `playos-platform-api` |
| **Runtime** | Lifecycle, package execution, OS integration | `playos-runtime` |
| **Shell** | Controller-first console UI (Raylib) | `playos-shell` |
| **Cloud** | Self-hostable backend services | `playos-cloud` |
| **Marketplace** | Publishing and distribution | `playos-marketplace` |
| **Tools** | Packaging, templates, diagnostics | `playos-tools` |
| **Samples** | Reference apps and tutorials | `playos-samples` |
| **Reference Devices** | Profiles, backends, bring-up | `playos-reference-devices` |
| **Foundation** | Governance, website, branding | `playos-foundation` |

The specification is the hub; every other repository implements part of it.
See the [Repository Map](12-repository-map.md) for details.

## Two kinds of targets

PlayOS distinguishes **Runtime Devices** (which run the full environment)
from **SDK Targets** (which support the Platform API only). This is a
core architectural idea, specified in the
[Target Model](../04-target-model/01-overview.md).

## Capability-based design

Applications discover features through capabilities rather than checking
platforms. This is what lets one codebase run across very different
devices. See the [Capability Model](../05-capability-model/01-overview.md).

## Reference implementation choices

The reference runtime is built on **Arch Linux** with a **wlroots/TinyWL**
compositor written in C++, and a **Raylib** shell running as a Wayland
client. These are reference-implementation decisions, recorded in
[ADR-0002](https://github.com/PlayOS-Foundation/playos-spec/blob/main/adr/0002-use-wlroots-tinywl-compositor.md)
and
[ADR-0003](https://github.com/PlayOS-Foundation/playos-spec/blob/main/adr/0003-arch-linux-reference-runtime-base.md).
The specification itself remains OS- and compositor-agnostic.
