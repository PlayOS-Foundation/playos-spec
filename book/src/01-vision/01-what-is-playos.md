# What Is PlayOS?

PlayOS is an **open console platform specification** and a set of reference
implementations. It defines how console-style games and applications are
built, run, packaged, distributed, and supported across many devices,
operating systems, and hardware profiles.

PlayOS is a **platform contract**, not a single product. The specification
is the source of truth; runtimes, SDKs, shells, cloud services, and
marketplaces are implementations of that contract.

## The one-sentence definition

> PlayOS is an open console platform specification and reference
> implementation designed to make developing console-style games simple,
> portable, and open.

## What PlayOS includes

PlayOS is made up of several parts, each specified in this book:

- a **Platform API** — the engine-agnostic contract applications call,
- a **Runtime** — the console operating environment for full devices,
- a **Shell** — the reference, controller-first console UI,
- a **Capability model** — how applications discover features,
- a **Package format** — how applications are distributed,
- a **Device model** — how hardware is described and ported,
- **Cloud and Marketplace** services — self-hostable ecosystem services,
- a **Certification** program — how devices, games, and stores conform,
- a **Governance** process — how the platform evolves.

## The layered picture

Applications sit on top of an engine, which sits on top of the PlayOS
Platform API, which is implemented by backends on each target:

```text
Game or Application
    ├── Raylib / SDL / Godot / Custom Engine
    └── PlayOS Platform API
            └── Backend Implementation
                    └── Runtime
                            └── Operating System
                                    └── Hardware
```

Raylib is the **reference** engine used by the shell, samples, and
tutorials — but the Platform API never depends on it. See
[Engine Agnostic](../02-platform-principles/02-engine-agnostic.md).

## Why it exists

Building a console-style experience on Linux today means wrestling with
evdev, libinput, PipeWire, Wayland, udev, and systemd directly. PlayOS
hides that behind a small, stable contract so that developers can build
games instead of operating systems, and players get a fast,
controller-first experience.

See also: [Mission](02-mission.md), [Non-Goals](04-non-goals.md), and the
[Platform Architecture Overview](../03-platform-architecture/01-overview.md).
