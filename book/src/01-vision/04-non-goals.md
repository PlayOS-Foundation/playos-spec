# Non-Goals

Defining what PlayOS is **not** keeps the platform coherent. The following
are explicit non-goals.

## PlayOS is not a game engine

PlayOS does not render graphics, simulate physics, or provide a scene
graph. Developers bring their own engine — Raylib, SDL, Godot, or a custom
one. The Platform API is engine-agnostic and must never become a
Raylib-only extension.

## PlayOS is not only a Linux distribution

A Linux-based runtime is the first full reference implementation, but the
platform is broader than one operating system. The specification must not
assume Linux where a capability query or backend abstraction is correct.

## PlayOS is not only a launcher

The shell is one component. PlayOS also defines APIs, packaging,
capabilities, cloud services, marketplace integration, device profiles,
certification, and tooling.

## PlayOS is not only a marketplace

The marketplace is a platform service, not the platform. PlayOS must be
usable with official, community, OEM, private, or fully self-hosted stores.

## PlayOS does not require its own cloud

Every cloud feature must have a local or self-hostable implementation.
PlayOS must never force dependence on a single central provider.

## PlayOS does not aim to replace Windows or SteamOS

The goal is not to compete head-to-head with general-purpose desktop
operating systems, but to provide an open, console-style platform and a
portable developer contract.

## PlayOS does not lock developers to one vendor

The specification is open so that anyone can implement it on a new platform
without permission. The reference implementations and hosted services are
one option, not the only option.

See also: [What Is PlayOS?](01-what-is-playos.md) and
[Mission](02-mission.md).
