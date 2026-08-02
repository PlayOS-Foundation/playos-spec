# Sprint 5 — Raylib-Powered PlayOS Shell

**Goal:** Build `playos-shell` as a hardware-accelerated Raylib Wayland client. It consumes the public `playos-platform-api` C ABI and presents a working controller-first game library UI on the ROG Ally.

**Primary Outcome:** The ROG Ally boots directly into a Raylib shell that shows a game list, responds to controller navigation (D-pad, A/B/X/Y), and remains persistently alive under `playos-compositor`.

**Prerequisites:** Sprint 4 complete — compositor owns the display; Sprint 3 complete — Platform API input contract defined.

---

## Key Deliverables

### Raylib PlayOS Backend — `rcore_playos.c`

Create a dedicated Raylib platform backend in `playos-platform-api` (or `playos-shell`) under `src/platforms/rcore_playos.c`.

**The backend adapts Raylib's Wayland support and integrates:**

| Feature | Implementation |
|---|---|
| Fullscreen Wayland surface | `xdg_toplevel` + `xdg_surface`, no decorations, no resize |
| EGL context creation | `eglCreateContext` on the wlroots/Wayland EGL platform |
| Frame callbacks | `wl_surface.frame` for v-sync pacing |
| PlayOS lifecycle events | Poll from `playos-runtime` transport channel; deliver as Raylib-style events |
| Trusted client identity | Set `PLAYOS_TRUSTED_SHELL=1` before connecting |
| Logical input mapping | Call `playos_input_get_controller_state()` each frame |
| Exit request | Send `Shutdown` over control IPC |
| PlayOS paths | Call `playos_storage_get_games_path()`, etc. |

**Unsupported desktop features (explicitly disabled or no-op):**
- Arbitrary window positioning
- Multiple desktop windows
- Window decorations
- File drag and drop
- Clipboard
- User-driven resize

### `playos-platform-api` — Additional API Groups

Define and implement the `libplayos` API groups needed by the shell this sprint:

**`include/playos/playos_lifecycle.h`**

```c
typedef enum {
    PLAYOS_LIFECYCLE_FOREGROUND,
    PLAYOS_LIFECYCLE_BACKGROUND,
    PLAYOS_LIFECYCLE_SUSPEND,
    PLAYOS_LIFECYCLE_RESUME,
    PLAYOS_LIFECYCLE_TERMINATE
} PlayOSLifecycleEvent;

/* Poll for the next lifecycle event. Returns 1 if an event is available. */
int playos_lifecycle_poll(PlayOSLifecycleEvent *event);
```

**`include/playos/playos_system.h`**

```c
/* API and OS version */
uint32_t playos_system_api_version(void);
const char *playos_system_os_version(void);
const char *playos_system_device_model(void);
```

**`include/playos/playos_storage.h`**

```c
/* Returns null-terminated path strings valid for the current game session. */
const char *playos_storage_get_games_path(void);
const char *playos_storage_get_saves_path(const char *game_id);
const char *playos_storage_get_cache_path(const char *game_id);
int64_t     playos_storage_free_bytes(void);
```

**`include/playos/playos_logging.h`**

```c
typedef enum { PLAYOS_LOG_DEBUG, PLAYOS_LOG_INFO, PLAYOS_LOG_WARN, PLAYOS_LOG_ERROR } PlayOSLogLevel;
void playos_log(PlayOSLogLevel level, const char *tag, const char *fmt, ...);
```

### `playos-shell` — Raylib Controller-First UI

Build the shell as a Raylib application using the `rcore_playos.c` backend.

**Initial UI screens:**

1. **Library screen** — Shows a scrollable grid of installed games
   - Reads game list from `playos_storage_get_games_path()` + manifest scan
   - Shows game name, icon (placeholder box if no icon), and version
   - D-pad navigates; A button selects

2. **Game selected screen** — Shows game detail
   - Game name, description (from manifest), storage size
   - "Launch" action (A button) — sends `LaunchGame` IPC (shows spinner, unimplemented response in this sprint)
   - "Back" action (B button) — returns to library

3. **System status bar** — Persistent footer
   - Battery level (from `playos_power_battery_percent()` — stub returning 100% for now)
   - Clock (system time)
   - PlayOS version

**Controller navigation rules:**
- D-pad navigates focus
- A button = confirm/select
- B button = back/cancel
- No mouse or keyboard input required for normal use

**Shell lifecycle behavior:**
- On `PLAYOS_LIFECYCLE_BACKGROUND`: stop rendering (call `SetTargetFPS(0)` or skip draw)
- On `PLAYOS_LIFECYCLE_FOREGROUND`: resume rendering
- On `PLAYOS_LIFECYCLE_TERMINATE`: clean shutdown

**Sample game stubs:** Include 3 fake game entries in `/data/games/` for visual testing:
- `com.playos.demo1/` with a `manifest.json`
- `com.playos.demo2/`
- `com.playos.demo3/`

### Buildroot Integration
- Add `package/playos-shell/` to `br2-external`
- Add Raylib to Buildroot (patch `rcore_playos.c` in, or build via CMake/Meson package)
- `playos-shell` depends on `libplayos` (from `playos-platform-api`)

---

## Acceptance Criteria

- [ ] ROG Ally boots and shell UI appears on screen without any input from user
- [ ] Game library grid shows 3 stub game entries
- [ ] D-pad navigation moves focus between games
- [ ] A button on a game shows the detail screen
- [ ] B button returns to library
- [ ] Shell renders at the display's native refresh rate (check via `fps` display)
- [ ] `playos-system.h`, `playos-input.h`, `playos-storage.h`, `playos-lifecycle.h`, `playos-logging.h` all compile with a C99 compiler
- [ ] Shell logs to `playos_log()` on startup, focus events, and navigation
- [ ] Shell survives 10 minutes of idle — no crash, no GPU hang
- [ ] QEMU headless: shell starts, Raylib initializes, no crash (may render blank/black)
- [ ] CI passes

---

## Repositories Primarily Involved

| Repo | Work |
|---|---|
| `playos-shell` | Raylib shell application, controller UI, screen layout |
| `playos-platform-api` | `rcore_playos.c` backend, `playos_lifecycle.h`, `playos_system.h`, `playos_storage.h`, `playos_logging.h` |
| `playos-refdistro` | Raylib Buildroot package, `playos-shell` package, stub game directories |
| `playos-spec` | Shell UX conventions, controller navigation rules |

---

## Testing Approach

- Physical ROG Ally for all visual/interaction tests
- Controller input test: navigate grid, enter/exit detail screen
- Lifecycle test: simulate `PLAYOS_LIFECYCLE_BACKGROUND` via IPC; verify rendering stops
- QEMU: shell starts without crash (headless, no visual)
- Memory: run Valgrind on host-native build for leaks during navigation

---

## Exit Gate

The ROG Ally boots directly into a Raylib shell showing a game library. Controller navigation works. Shell uses the `libplayos` public API for input, storage, lifecycle, and logging.

*Previous: [Sprint 4](Sprint-4.md) | Next: [Sprint 6](Sprint-6.md)*
