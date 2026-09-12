# `playos-overlay` Specification

> **Repository:** `playos-overlay` (or future multi-surface backend in `playos-shell`)  
> **Role:** Trusted system overlay — quick menu, notifications, power management UI  
> **Language:** C (Raylib)  
> **Cross-references:** [architecture.md](architecture.md) §7.4, §8.3, [wayland-protocol.md](wayland-protocol.md), [sprints/Sprint-7.md](sprints/Sprint-7.md)

---

## Responsibilities

| Owns | Does NOT own |
|---|---|
| Quick menu presentation | Game logic |
| Volume and brightness HUD | Shell game library |
| Power menu (shutdown, restart, sleep) | Process supervision |
| Notifications | DRM/KMS policy |
| Virtual keyboard (future) | IPC protocol definitions |
| Performance profile selector | Save management |

---

## Why a Separate Process

Standard Raylib is most comfortable with one native EGL surface per process. The overlay and shell both require independently rendered fullscreen surfaces. A separate trusted process keeps the overlay isolated from shell state and allows it to appear above any surface — including a crashed or frozen game — without depending on the shell being responsive.

The overlay may later be merged into a multi-surface `playos-shell` backend if Raylib multi-surface support improves.

---

## Overlay Lifecycle

```
playos-init spawns and supervises playos-overlay at boot
    │
    ├── Overlay creates xdg_toplevel surface (transparent background)
    ├── Registers as trusted overlay via playos_manager_v1::register_overlay
    ├── Receives about_to_show / about_to_hide events from compositor
    │
[System button pressed — compositor sends about_to_show]
    │
    ├── Overlay renders quick menu frame
    ├── Sends surface_ready to compositor
    ├── Compositor maps overlay above game (z-order 3)
    ├── Input routed to overlay
    │
[User presses Resume or game-related action]
    │
    ├── Overlay sends request_dismiss to compositor
    ├── Compositor hides overlay, returns focus to game
    └── Overlay receives about_to_hide; clears its surface
```

The overlay is **always alive** but only visible when the compositor maps it. Its rendering is stopped when hidden.

**Input ownership.** The overlay only reacts to the gamepad while it is visible. When hidden it still drains its own evdev fd but discards every decoded action, and the compositor sends `about_to_hide` whenever it clears overlay visibility — so the game keeps its buttons during play (notably B, which the overlay maps to Quit only while shown).

---

## Screens

### Quick Menu (default when overlay is shown)

```
┌────────────────────────────────────────────┐
│  [Game name]             [battery] [time]  │
├────────────────────────────────────────────┤
│                                            │
│   ▶  Resume Game                           │
│      Quit Game                             │
│                                            │
│   Volume:  ████████░░  75%                 │
│   Profile: [Balanced ▼]                    │
│                                            │
│   CPU: 72°C  GPU: 68°C                     │
│                                            │
└────────────────────────────────────────────┘
```

Navigation (implemented): A = Resume, B = Quit game, D-pad Up/Down = volume step, D-pad Left/Right = performance profile, SELECT = power menu. Within the power menu: Up/Down = move cursor, A = confirm, B = back. Within the profile selector: Left/Right = change, A = apply, B = back. The d-pad is decoded from both `ABS_HAT0X/ABS_HAT0Y` (xpad / hid-asus on the ROG Ally) and `BTN_DPAD_*`.

### Power Menu (accessed from Quick Menu)

```
┌──────────────────────────────┐
│       Power Options          │
├──────────────────────────────┤
│   Sleep                      │
│   Restart                    │
│   Shut Down                  │
└──────────────────────────────┘
```

Opened with SELECT from the quick menu; Up/Down moves the cursor, A confirms, B returns to the quick menu.

- **Sleep** → `Suspend` IPC. `playos-init` delivers `PLAYOS_LIFECYCLE_SUSPEND` to the running game, writes `mem` to `/sys/power/state` (S3 suspend-to-RAM), then delivers `PLAYOS_LIFECYCLE_RESUME` after wake. If `/sys/power/state` is unavailable it logs `suspend unavailable` and is a no-op.
- **Restart** → orderly reboot (`playos_shutdown(s, 1)` → `reboot(RB_AUTOBOOT)`).
- **Shut Down** → orderly power-off (`playos_shutdown(s, 0)` → `reboot(RB_POWER_OFF)`).

Factory Reset is not offered from the overlay; it lives in the shell Settings and the recovery menu.

---

## Notification System

Notifications are short messages shown at the bottom of the screen (when not in quick menu mode).

**Types:**
- `INFO` — blue badge, auto-dismiss after 3 seconds
- `WARNING` — yellow badge, auto-dismiss after 5 seconds
- `ERROR` — red badge, requires A-button dismiss

**Sources:**
- Game crash recovery: "Game exited unexpectedly"
- Thermal state change: "Performance reduced — device is hot"
- Low battery: "Battery low: 15%"
- Update available: "System update ready"

Notifications are queued; at most one is shown at a time.

---

## Volume Control

Volume is adjusted via the overlay:
- D-pad Left/Right on the volume slider: ±5% per step
- Calls `playos_audio_set_master_volume(new_volume)`
- Mute toggle: L1 button
- Visual: filled/empty bar + percentage text

---

## Performance Profile Selector

Displays `PLAYOS_PERF_BALANCED`, `PLAYOS_PERF_POWER_SAVE`, `PLAYOS_PERF_PERFORMANCE` in a dropdown.

- D-pad Left/Right to cycle options
- A to confirm → calls `playos_power_request_profile()`
- If profile is denied (thermal override): shows "Performance limited — device is hot" notification

---

## Trusted Client Identity

The overlay sets `PLAYOS_TRUSTED_OVERLAY=1` in its environment. The compositor verifies this at connection time and assigns the `playos_overlay_v1` role.

The overlay may also use the `playos-runtime` restricted client to:
- Send `TerminateGame` (Quit action)
- Send `Shutdown` / `Reboot` (power menu)
- Send `FactoryReset` (with appropriate flags)
- Send `SetPerfProfile`

---

## Scene Integration

The compositor renders a dimming layer between the game and the overlay surface:

```
z-order:
  1. game surface (visible, not focused)
  2. compositor-rendered dimming layer (rgba(0,0,0,0.4))
  3. overlay surface (transparent background; UI elements are opaque)
```

The overlay's Wayland surface uses a transparent background (`EGL_ALPHA_SIZE = 8`, pre-multiplied alpha). Only drawn UI elements are opaque.

---

## Build

```cmake
# playos-overlay/CMakeLists.txt
find_package(playos-platform-api REQUIRED)
find_package(raylib REQUIRED)

target_sources(playos-overlay PRIVATE
    src/main.c
    src/screens/quick_menu.c
    src/screens/power_menu.c
    src/ui/notification.c
    src/ui/volume_bar.c
    src/ui/profile_selector.c
    src/ui/factory_reset.c
)
target_link_libraries(playos-overlay playos raylib)
```
