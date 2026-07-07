# Engine Agnostic

The PlayOS Platform API **must never depend on a single game engine.**

PlayOS is a platform SDK, not a game engine. It provides platform services
— input, storage, display, audio, power, lifecycle, cloud, marketplace —
not rendering or gameplay systems.

## The contract

Applications use an engine for rendering and the PlayOS Platform API for
platform services. The two are independent:

```text
Game or Application
    ├── Raylib / SDL / Godot / Custom Engine   (rendering, gameplay)
    └── PlayOS Platform API                    (platform services)
```

This is the wrong relationship:

```text
Game or Application
    └── Raylib-only PlayOS extension           (rejected)
```

## Raylib is the reference, not a requirement

Raylib is important to PlayOS because it is small, fast, open, and
approachable. It is used for:

- the reference shell,
- the official samples and tutorials,
- the starter templates.

But the Platform API must remain usable from SDL, Godot, bgfx, or a custom
engine. If Raylib were replaced tomorrow, applications written against the
PlayOS Platform API would still be valid.

## Why this matters

Modern console SDKs work this way: the platform holder exposes an API, not
an engine. Nintendo, Sony, and Microsoft do not mandate a rendering engine.
Keeping PlayOS engine-agnostic means the platform can outlive any single
engine and welcome the widest range of developers.

## Practical guidance

- Public Platform API headers must not include engine headers.
- Examples may use Raylib, but the API they demonstrate must be
  engine-neutral.
- Engine-specific helpers belong in an engine integration kit (see
  [Engine Integration](../07-engine-integration/01-overview.md)), not in
  the Platform API.

See also: [Specification First](01-specification-first.md) and
[What Is PlayOS?](../01-vision/01-what-is-playos.md).
