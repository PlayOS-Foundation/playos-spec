# Reference Shell

The **PlayOS Shell** is the reference, controller-first console UI. It is
built with **Raylib** and runs as a **Wayland client** of the PlayOS
compositor. It is the player's home screen and the launcher for games.

## Role

- Present the player-facing console experience (home, library, settings,
  marketplace, and power).
- Launch applications through the runtime and return to the foreground when
  they exit.
- Remain replaceable: the shell is a client, not the compositor, so
  alternative shells can exist.

## Technology

- **Raylib** for rendering and input polling.
- **Wayland client** surface composited by the PlayOS compositor (see
  [Compositor Model](../08-runtime-architecture/05-compositor-model.md)).
- Uses the PlayOS Platform API for input, storage, lifecycle, battery,
  network, and Bluetooth. The shell MUST NOT perform direct OS calls
  (no `#ifdef __linux__`, no `getifaddrs`, no sysfs reads). All hardware
  access goes through Platform API backends.

## Constraints

- The shell MUST NOT depend on desktop components (no KDE, GNOME, taskbar,
  or visible terminal, no mouse cursor by default).
- The shell MUST be fully operable with a controller (see
  [Controller-First Navigation](03-controller-first-navigation.md)).
- The shell MUST tolerate services that are not yet ready (audio, network,
  library) and update live as they come online.
- The shell MUST NOT contain OS-specific branching (`#ifdef`) in application
  code. All platform differences are hidden in Platform API backends.

## Screen architecture

The shell is structured as a **screen stack**. Each distinct UI view is a
screen; screens are pushed onto the stack to navigate forward and popped
to go back. Only the topmost screen receives input and draws itself;
all screens below are paused.

```text
ScreenStack
  │
  ├─ LibraryScreen        ← default (bottom of stack)
  ├─ OverlayScreen        ← pushed on Home button
  ├─ ConfigScreen         ← pushed from Overlay → Settings
  │   ├─ WiFiScreen
  │   └─ BluetoothScreen
  └─ MarketplaceScreen    ← pushed from Library → Store
```

The `IScreen` interface every screen implements:

```cpp
class IScreen {
public:
    virtual void OnEnter() {}           // called when pushed
    virtual void OnExit()  {}           // called when popped
    virtual void Update(float dt) = 0;  // input + logic
    virtual void Draw(int W, int H) = 0;
};
```

Navigation rules:
- **A / Confirm** — push the target screen.
- **B / Back** — pop the current screen. The LibraryScreen (bottom) never pops.
- **Home** — push the OverlayScreen from any depth.
- **Back on Overlay** — pop back to whichever screen was below.

## Status bar

A persistent `StatusBar` component is drawn above all screens on every
frame. It is not a screen — it cannot be navigated to or from. It shows:

| Indicator | Source | Behaviour |
|---|---|---|
| Battery level + icon | `PlayOS::Battery` | Hidden when `Capability::Battery` absent |
| WiFi icon | `PlayOS::Network::GetWiFiState()` | Green/yellow/dim by state |
| Bluetooth icon | `PlayOS::Capabilities::Has(BluetoothPresent)` | Blue if present, dim if absent |
| IP address | `PlayOS::Network::PrimaryIP()` | Green, bottom-right |

Status bar indicators MUST be polled periodically (suggested: every 5 s)
rather than re-queried every frame, to avoid excessive syscalls.

## Icon rendering

The shell uses the **Remixicon** icon font (`remixicon.ttf`) for status
indicators. The font file is bundled in the reference distro at
`/usr/share/playos/fonts/remixicon.ttf`. The shell MUST fall back
gracefully to ASCII letters (`W`, `B`, `%`) when the font file is absent
(e.g. on a development machine without the ISO assets).

## First milestone (definition of done — v0.1 complete)

```text
[x] Raylib shell runs on the Arch-based image
[x] ROG Ally controls navigate the library
[x] A launches a sample game
[x] Game exits and returns to the shell
[x] Battery, WiFi, BT status visible via Remixicon icons
[x] IP address shown (for SSH debugging)
[x] No OS-specific ifdefs in shell source
```

See: [Overview](01-overview.md),
[Game Switching](../08-runtime-architecture/11-game-switching.md),
[Reference Shell ROADMAP](https://github.com/PlayOS-Foundation/playos-shell/blob/main/ROADMAP.md).
