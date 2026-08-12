# `playos-shell` Specification

> **Repository:** `playos-shell`  
> **Role:** Persistent controller-first console UI; trusted Wayland client  
> **Language:** C (Raylib)  
> **Cross-references:** [architecture.md](architecture.md) §7.3, [platform-api.md](platform-api.md), [Sprint-5.md](Sprint-5.md), [Sprint-6.md](Sprint-6.md)

---

## Responsibilities

| Owns | Does NOT own |
|---|---|
| Persistent console UI | Process supervision |
| Controller-first navigation | DRM/KMS or Wayland protocol |
| Game discovery and metadata presentation | IPC protocol definitions |
| User-facing launch, resume, quit, crash flows | Game installation |
| Settings and status screens | Save data management |
| Launch requests via restricted control IPC | Hardware driver details |
| Rendering with Raylib PlayOS backend | |
| Preserving UI state while a game runs | |

---

## Screen Architecture

```
Library Screen (default)
    │  A button → select game
    ▼
Game Detail Screen
    │  A button → launch
    │  B button → back
    ▼
Launching Screen (spinner)
    │  game becomes foreground → shell backgrounds
    │
    [game running — shell alive but rendering stopped]
    │
    [PLAYOS_LIFECYCLE_FOREGROUND → shell returns]
    ▼
Library Screen (restored position)
    │  (optionally shows post-crash notification)
```

Additional screens:
- **Settings Screen** — display brightness (stub), audio (deferred), system info, update check
- **System Update Screen** — current version, download/apply progress

---

## Rendering Lifecycle

| Shell state | Rendering behavior |
|---|---|
| Foreground | Full render at display refresh rate |
| Game launching | Show spinner; reduce to 10 FPS |
| Game is foreground | **Stop rendering** — `SetTargetFPS(0)` or skip draw call |
| Game backgrounded (overlay visible) | Shell stays stopped; overlay renders |
| Returning to foreground | Resume rendering; restore previous screen and cursor position |

The shell surface **remains alive** (mapped as a Wayland surface) at all times. Only rendering is stopped, not the process.

---

## Controller Navigation Rules

| Input | Action |
|---|---|
| D-pad Up/Down | Move focus up/down in a list or grid |
| D-pad Left/Right | Move focus left/right in a grid |
| A button | Confirm / select focused item |
| B button | Back / cancel |
| Start | Open settings screen |
| Select | Toggle sort/filter (library screen) |
| L1 / R1 | Page left/right (future) |
| System button | Never reaches shell — intercepted by compositor |

Mouse and keyboard are not required for normal shell use. They may be used for development convenience.

---

## Game Discovery

The shell scans `playos_storage_get_games_root()` on startup and on explicit refresh:

```c
// Pseudo-code
const char *games_root = playos_storage_get_games_root();
DIR *dir = opendir(games_root);
while ((entry = readdir(dir))) {
    char manifest_path[PATH_MAX];
    snprintf(manifest_path, sizeof(manifest_path),
             "%s/%s/manifest.json", games_root, entry->d_name);
    GameManifest manifest;
    if (parse_manifest(manifest_path, &manifest) == 0) {
        game_list_add(&games, &manifest);
    } else {
        PLAYOS_LOG_W("shell", "Skipping invalid manifest: %s", manifest_path);
    }
}
sort_games_by_name(&games);
```

Game icons are loaded as Raylib `Texture2D` from `<install_path>/assets/icon.png`. A placeholder texture is used when no icon is present.

---

## Launch Flow (Shell Side)

```
User selects "Launch" on game detail screen
    │
    ├── Shell sends LaunchGame via control IPC (playos-runtime client)
    │
    ├── Shell transitions to "Launching" screen (spinner + game name)
    │
    ├── Receives GameStarted event (async) — game PID is known
    │
    └── Receives PLAYOS_LIFECYCLE_BACKGROUND
          │
          └── Shell stops rendering; game is now foreground
```

Error handling:
- `LaunchGameError(already_running)` — show "A game is already running" notification
- `LaunchGameError(invalid_manifest)` — show "This game cannot be launched" notification
- Timeout (no `GameStarted` within 10s) — show "Launch failed, please try again"

---

## Post-Game Return Flow

When the game exits or crashes, the compositor delivers `PLAYOS_LIFECYCLE_FOREGROUND` to the shell:

```
Shell receives PLAYOS_LIFECYCLE_FOREGROUND
    │
    ├── Resume rendering
    ├── Restore previous library position and selection
    │
    └── If crashed == true:
          Show notification overlay: "Game exited unexpectedly"
          Options: "Restart" | "Back to Library"
```

---

## Status Bar

Persistent footer visible on all shell screens:

| Element | Source |
|---|---|
| Battery % + charging icon | `playos_power_get_info()` |
| Thermal indicator (color) | `playos_power_get_info().thermal_state` |
| System time | `clock_gettime(CLOCK_REALTIME)` |
| PlayOS version | `playos_system_os_version()` |

Updated every 30 seconds for battery/thermal; every 1 second for clock.

---

## Trusted Client Identity

The shell sets `PLAYOS_TRUSTED_SHELL=1` in its own environment before connecting to the Wayland display. The compositor verifies this at connection time and assigns the `playos_shell_v1` role.

The shell uses the `playos-runtime` restricted control client library to:
- Connect to `/run/playos/control.sock`
- Send `LaunchGame`, `QueryStatus`, `Shutdown`, `Reboot`, `FactoryReset`
- Receive async events: `GameStarted`, `GameExited`, `ThermalStateChanged`

---

## Raylib PlayOS Backend (`rcore_playos.c`)

Raylib 6.0 is **active** (Sprint 5.5). It is vendored into
`playos-shell/external/raylib` (pinned by `RAYLIB_COMMIT` in
`versions.lock`) and built as a static library with the custom
`PLATFORM_PLAYOS` backend (`external/raylib/src/platforms/rcore_playos.c`).

The backend implements:
- Wayland connection and `wl_compositor`/`xdg_wm_base`/`playos_manager_v1` globals
- Fullscreen `xdg_toplevel` surface — no decorations, no resize
- `wl_egl_window` + EGL/GLES2 context (`eglBindAPI(EGL_OPENGL_ES_API)`), made current before raylib's `rlgl` init
- Frame callbacks for v-sync pacing + `eglSwapBuffers` in `SwapScreenBuffer()`

Rendering is Raylib-only; Raylib is **not** the input path. Controller input
stays shell-owned direct evdev (`src/input.c`) so reserved SYSTEM/QUICK_MENU
buttons survive. `PollInputEvents()` in the backend only resets raylib's
internal input state. Lifecycle events are polled in `main.c` via
`playos_lifecycle_poll()` — suspend/background skips `BeginDrawing`/
`EndDrawing`, and TERMINATE exits cleanly (`running = false`, no bare
`exit()`).

---

## Build

```cmake
# playos-shell/CMakeLists.txt (PLAYOS_SHELL_USE_RAYLIB=ON)
add_subdirectory(external/raylib)   # vendored Raylib 6.0, PLATFORM=PlayOS
find_library(PLAYOS_LIB playos ...) # libplayos from playos-platform-api

target_sources(playos-shell PRIVATE
    src/main.c
    src/input.c
    src/screen_home.c
    src/screen_library.c
    src/screen_game_detail.c
    src/screen_settings.c
    src/render_util.c
)
target_link_libraries(playos-shell PRIVATE raylib ${PLAYOS_LIB} m)
```
