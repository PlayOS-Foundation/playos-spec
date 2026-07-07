# Reference Shell

The **PlayOS Shell** is the reference, controller-first console UI. It is
built with **Raylib** and runs as a **Wayland client** of the PlayOS
compositor. It is the player's home screen and the launcher for games.

## Role

- Present the player-facing console experience (home, library, and later
  store, settings, downloads, power).
- Launch applications through the runtime and return to the foreground when
  they exit.
- Remain replaceable: the shell is a client, not the compositor, so
  alternative shells can exist.

## Technology

- **Raylib** for rendering and input polling.
- **Wayland client** surface composited by the PlayOS compositor (see
  [Compositor Model](../08-runtime-architecture/05-compositor-model.md)).
- Uses the PlayOS Platform API for input, storage, and lifecycle; it does
  not talk to evdev/libinput/Wayland directly for platform services.

## Constraints

- The shell MUST NOT depend on desktop components (no KDE, GNOME, taskbar,
  or visible terminal, no mouse cursor by default).
- The shell MUST be fully operable with a controller (see
  [Controller-First Navigation](03-controller-first-navigation.md)).
- The shell MUST tolerate services that are not yet ready (audio, network,
  library) and update live as they come online.

## Minimal shell (vertical slice)

The proof-of-concept shell is intentionally tiny:

```text
PlayOS Shell (Raylib, Wayland client)
  ├─ render a simple game list
  ├─ controller navigation (D-Pad / stick, A = launch, B = back)
  ├─ ask the runtime to launch the selected game
  └─ on Home / game exit, regain foreground and redraw
```

No store, cloud, themes, or settings are required for the first slice — just
enough to prove the console loop end to end on the ROG Ally.

## First milestone (definition of done)

```text
Raylib shell runs on the Arch-based image
ROG Ally controls navigate the UI
A launches one demo game
Home returns from the game to the shell
```

See: [Overview](01-overview.md),
[Game Switching](../08-runtime-architecture/11-game-switching.md),
[What Is PlayOS?](../01-vision/01-what-is-playos.md).
