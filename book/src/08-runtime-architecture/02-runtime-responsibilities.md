# Runtime Responsibilities

The PlayOS runtime is the software layer that sits between the operating
system and the running application.  It owns the display, manages the
application lifecycle, routes input, and enforces platform contracts so
that applications need not care which hardware they are running on.

## Core responsibilities

### 1. Display ownership

The runtime takes exclusive ownership of the display via DRM/KMS at boot.
No display server, desktop environment, or window manager runs alongside it.
The compositor (see [Compositor Model](05-compositor-model.md)) is the sole
DRM master; all visual output goes through it.

### 2. Application launch and supervision

The runtime launches applications on behalf of the shell using
`PlayOS::Runtime::LaunchAndWait()`.  Each application runs in its own
process group.  The runtime:

- spawns the process (`fork`/`exec` on POSIX, `CreateProcess` on Windows),
- waits for it to exit,
- returns the exit code to the shell.

Abnormal exits (crash, signal, non-zero code) are reported as a
`LaunchResult` to the shell.  The shell decides whether to surface an error
or silently return to the library.

### 3. Input routing

Input devices are owned by the runtime compositor via libinput.  The
compositor routes keyboard and pointer events to the Wayland-focused surface.
Controller input is delivered directly to applications through the PlayOS
Platform API evdev backend, bypassing Wayland entirely (see
[Input Routing](10-input-routing.md)).

The **Home** button is intercepted by the compositor before reaching any
application.  It raises the system overlay regardless of what is running.

### 4. Game switching

When an application exits, the compositor returns focus and the display to
the shell.  The transition is synchronised through the Wayland surface
lifecycle: the compositor waits for the application surface to be destroyed
before reconfiguring the shell surface to full resolution (see
[Game Switching](11-game-switching.md)).

### 5. System overlay

The runtime hosts a persistent overlay surface above all application content.
The overlay is raised on Home-button press and shows battery level, WiFi
status, volume, and a Return-to-Shell action.  The overlay is a Wayland
client of the compositor; it cannot be obscured by the running game.

> **Stage 1 note:** The overlay is not yet implemented.  Home-button press
> is handled inside the shell process as a local menu.

### 6. Async platform services

Services that are not on the critical display path start after the first
shell frame is visible:

| Service | Responsibility |
|---|---|
| `playos-audio` | PipeWire session + WirePlumber policy |
| `playos-network` | NetworkManager + WiFi |
| `playos-bluetooth` | BlueZ daemon |
| `playos-library` | Local game library indexing |
| `playos-update` | Background update check |

These services MUST NOT block `playos-compositor.service` from starting.
Any service that cannot satisfy the [boot budget](../08-runtime-architecture/03-boot-model.md)
belong in `playos-async.target`.

### 7. Suspend and resume

On devices with a sleep button or lid, the runtime handles suspend/resume:

- Notifies the running application via the lifecycle API (`Lifecycle::Update()`
  returns a suspend signal).
- Waits for the application to flush save data (bounded timeout).
- Signals the compositor to drop the DRM master.
- Triggers system suspend via `systemd-logind`.
- On resume, the compositor re-acquires DRM and repaints the shell or
  returns the application surface.

> **Stage 1 note:** Suspend/resume is not yet implemented.

## What the runtime is NOT responsible for

- **Rendering.** The compositor composites surfaces; applications own their own rendering.
- **Game engine internals.** Audio mixing, physics, scripting are in the application.
- **Marketplace / cloud.** Those services connect to the platform APIs but
  are not part of the runtime process.  They run in separate processes under
  `playos-async.target`.
