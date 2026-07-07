# Compositor Model

The PlayOS reference runtime uses its **own compositor**, built on
**wlroots**, starting from the **TinyWL** reference and written in **C++**.
The compositor owns the display (DRM/KMS); the shell and games are Wayland
clients. This decision is recorded in
[ADR-0002](https://github.com/PlayOS-Foundation/playos-spec/blob/main/adr/0002-use-wlroots-tinywl-compositor.md).

## Responsibilities

The compositor is responsible for:

- owning DRM/KMS and page-flipping to the display,
- managing fullscreen application surfaces,
- routing input to the focused surface (see [Input Routing](10-input-routing.md)),
- switching between the shell and a running game (see [Game Switching](11-game-switching.md)),
- hosting the system overlay surface,
- supporting XWayland for non-Wayland applications.

## Client model

```text
PlayOS Compositor (wlroots / TinyWL, C++)
    ├── PlayOS Shell   (Raylib, Wayland client)
    ├── Running Game   (Wayland or XWayland client)
    └── Overlay        (system surface)
```

The shell is **not** the compositor. It is a normal client that the
compositor composites like any other, which keeps the shell replaceable and
the compositor focused on display and input.

## Why wlroots/TinyWL

wlroots provides mature DRM/KMS, libinput, output management, and XWayland
support. TinyWL is a ~1000-line starting point small enough to understand
and evolve. The alternative approaches — Gamescope (not owned by PlayOS) and
bare Raylib DRM (reinvents input routing, overlay, and switching) — were
rejected in ADR-0002.

## Staged evolution

```text
Stage 1: TinyWL + fullscreen-only + controller input
Stage 2: + launcher IPC, Home button, game switching, overlay, XWayland
Stage 3: + performance HUD, suspend/resume, brightness/volume
```

## Vertical slice note

The proof-of-concept targets **Stage 1**: a TinyWL-based compositor that
brings up the display on the Ally, composites the Raylib shell fullscreen,
routes controller input to it, and can give the display to a launched game
and take it back on exit. Overlay and suspend/resume are deferred.

See: [Boot Model](03-boot-model.md), [Game Switching](11-game-switching.md),
[Reference Shell](../09-shell-and-ux/16-reference-shell.md).
