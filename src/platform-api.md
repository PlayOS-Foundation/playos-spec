# PlayOS Platform API Specification

> **Authoritative repository:** `playos-platform-api`  
> **Current ABI version:** 1  
> **SONAME:** `libplayos.so.0`  
> **Cross-references:** [architecture.md](architecture.md) §7.6, [security-model.md](security-model.md)

---

## Table of Contents

1. [Purpose and Scope](#1-purpose-and-scope)
2. [ABI Stability Policy](#2-abi-stability-policy)
3. [Versioning](#3-versioning)
4. [Header Organization](#4-header-organization)
5. [API Groups](#5-api-groups)
   - 5.1 [System](#51-playos_systemh)
   - 5.2 [Lifecycle](#52-playos_lifecycleh)
   - 5.3 [Input](#53-playos_inputh)
   - 5.4 [Display](#54-playos_displayh)
   - 5.5 [Storage](#55-playos_storageh)
   - 5.6 [Audio](#56-playos_audioh)
   - 5.7 [Power](#57-playos_powerh)
   - 5.8 [Logging](#58-playos_loggingh)
6. [Backend Architecture](#6-backend-architecture)
7. [C++ and Engine Wrappers](#7-c-and-engine-wrappers)
8. [What the API Must Never Expose](#8-what-the-api-must-never-expose)
9. [Change Process](#9-change-process)

---

## 1. Purpose and Scope

`playos-platform-api` owns the **public, engine-agnostic PlayOS application contract**. It is the only interface that games, the shell, and tools may use to interact with PlayOS capabilities.

**`libplayos` hides:**
- Linux kernel internals
- DRM/KMS device handles
- Wayland compositor internals
- `playos-runtime` private IPC endpoints
- Filesystem paths other than the safe storage API

**`libplayos` exposes:**
- Lifecycle events (foreground, background, suspend, terminate)
- Assigned storage paths (install, save, cache)
- Device and capability information
- Logical input state
- Structured logging
- Audio, display, and power queries that are safe for applications
- Narrow requests for approved system actions (performance profile)

---

## 2. ABI Stability Policy

The C ABI is the authoritative compatibility boundary.

### What is ABI-stable (safe to add in a minor version):
- New enum values at the **end** of an existing enum
- New functions with new names
- New struct types
- New header files

### What is ABI-breaking (requires a major version bump):
- Removing or renaming any public symbol
- Changing a function signature
- Changing struct field layout, size, or order
- Redefining enum values
- Changing the semantic meaning of any return value

### Language compatibility
The C ABI must be consumable from: C, C++, Rust, Zig, and any language with a C FFI. All public headers must be valid C99 and C++11.

```c
/* All public headers include this guard */
#ifdef __cplusplus
extern "C" {
#endif
/* ... declarations ... */
#ifdef __cplusplus
}
#endif
```

---

## 3. Versioning

```c
/* include/playos/playos.h — master version header */
#define PLAYOS_API_VERSION_MAJOR  0
#define PLAYOS_API_VERSION_MINOR  3
#define PLAYOS_API_VERSION_PATCH  0
#define PLAYOS_API_VERSION        1  /* integer for runtime checks */
```

**Runtime version query:**
```c
uint32_t playos_system_api_version(void);  /* returns PLAYOS_API_VERSION */
```

**SONAME policy:**
- `libplayos.so.0` — covers all v0.x.y releases
- `libplayos.so.1` — next breaking release

**Breaking changes require:**
1. RFC filed in `playos-spec`
2. ADR documenting the decision and migration path
3. Major SONAME bump
4. Migration guide published before release

---

## 4. Header Organization

```
include/playos/
    playos.h                  # master include — pulls in all groups
    playos_system.h           # device and OS information
    playos_lifecycle.h        # lifecycle events
    playos_input.h            # controller input
    playos_display.h          # display information
    playos_storage.h          # save, cache, and install paths
    playos_audio.h            # audio state and volume
    playos_power.h            # battery, thermal, performance profiles
    playos_logging.h          # structured logging
```

Convention: every symbol is prefixed `playos_` (functions and variables) or `PLAYOS_` (constants and macros).

---

## 5. API Groups

### 5.1 `playos_system.h`

Device and platform information. Read-only.

```c
/* API version this runtime implements. */
uint32_t    playos_system_api_version(void);

/* Null-terminated version string, e.g. "0.1.0". */
const char *playos_system_os_version(void);

/* Null-terminated device model string, e.g. "ROG Ally (2023)". */
const char *playos_system_device_model(void);

/* CPU and GPU description strings. */
const char *playos_system_cpu_description(void);
const char *playos_system_gpu_description(void);

/* Total and available RAM in bytes. */
uint64_t    playos_system_total_memory_bytes(void);
uint64_t    playos_system_available_memory_bytes(void);

/* BCP 47 locale string, e.g. "en-US". */
const char *playos_system_locale(void);
```

All returned `const char *` strings are valid for the lifetime of the process. Never `free()` them.

---

### 5.2 `playos_lifecycle.h`

Lifecycle events delivered by `playos-runtime` via a fd established at launch.

```c
typedef enum {
    PLAYOS_LIFECYCLE_FOREGROUND,   /* game is now the active foreground surface */
    PLAYOS_LIFECYCLE_BACKGROUND,   /* game is hidden; should pause and reduce CPU */
    PLAYOS_LIFECYCLE_SUSPEND,      /* system is suspending; save state immediately */
    PLAYOS_LIFECYCLE_RESUME,       /* system resumed from suspend */
    PLAYOS_LIFECYCLE_TERMINATE     /* ordered shutdown; clean up and exit promptly */
} PlayOSLifecycleEvent;

/* Non-blocking. Returns 1 if an event was written to *event, 0 if none pending,
   -1 on error (fd closed or invalid). */
int playos_lifecycle_poll(PlayOSLifecycleEvent *event);

/* Blocking variant — waits up to timeout_ms milliseconds (-1 = indefinite). */
int playos_lifecycle_wait(PlayOSLifecycleEvent *event, int timeout_ms);

/* Returns the underlying fd for use with poll(2) / select(2). */
int playos_lifecycle_fd(void);
```

**Expected game behavior:**

| Event | Required action |
|---|---|
| `FOREGROUND` | Resume rendering and input processing |
| `BACKGROUND` | Pause gameplay, stop normal input, lower/mute audio, reduce FPS to 0 |
| `SUSPEND` | Flush save data immediately; return within 500ms |
| `RESUME` | Restore state; resume rendering |
| `TERMINATE` | Save state, release resources, call `exit(0)` within 2 seconds |

---

### 5.3 `playos_input.h`

Logical controller state. Hardware-agnostic.

```c
typedef enum {
    PLAYOS_BUTTON_SOUTH       = (1 << 0),   /* A on Xbox layout */
    PLAYOS_BUTTON_EAST        = (1 << 1),   /* B */
    PLAYOS_BUTTON_WEST        = (1 << 2),   /* X */
    PLAYOS_BUTTON_NORTH       = (1 << 3),   /* Y */
    PLAYOS_BUTTON_START       = (1 << 4),
    PLAYOS_BUTTON_SELECT      = (1 << 5),
    PLAYOS_BUTTON_DPAD_UP     = (1 << 6),
    PLAYOS_BUTTON_DPAD_DOWN   = (1 << 7),
    PLAYOS_BUTTON_DPAD_LEFT   = (1 << 8),
    PLAYOS_BUTTON_DPAD_RIGHT  = (1 << 9),
    PLAYOS_BUTTON_L1          = (1 << 10),
    PLAYOS_BUTTON_R1          = (1 << 11),
    PLAYOS_BUTTON_L3          = (1 << 12),  /* left stick click */
    PLAYOS_BUTTON_R3          = (1 << 13),  /* right stick click */
    /* PLAYOS_BUTTON_SYSTEM and PLAYOS_BUTTON_QUICK_MENU are reserved
       and are never delivered to games. */
} PlayOSButton;

typedef enum {
    PLAYOS_AXIS_LEFT_X        = 0,
    PLAYOS_AXIS_LEFT_Y        = 1,
    PLAYOS_AXIS_RIGHT_X       = 2,
    PLAYOS_AXIS_RIGHT_Y       = 3,
    PLAYOS_AXIS_LEFT_TRIGGER  = 4,
    PLAYOS_AXIS_RIGHT_TRIGGER = 5,
    PLAYOS_AXIS_COUNT         = 6
} PlayOSAxis;

typedef struct {
    uint32_t buttons;                /* bitmask of PlayOSButton flags */
    float    axes[PLAYOS_AXIS_COUNT]; /* sticks: [-1.0, 1.0]; triggers: [0.0, 1.0] */
    uint64_t timestamp_us;           /* microseconds since system boot */
} PlayOSControllerState;

/* Returns 1 if the primary controller is connected. */
int playos_input_controller_connected(void);

/* Fills *state with the current controller snapshot. Returns 0 on success. */
int playos_input_get_controller_state(PlayOSControllerState *state);

/* Helper: returns non-zero if the given button flag is set in state->buttons. */
static inline int playos_input_button_down(const PlayOSControllerState *state,
                                           PlayOSButton button) {
    return (state->buttons & (uint32_t)button) != 0;
}
```

---

### 5.4 `playos_display.h`

Display information. Read-only; games do not configure the display.

```c
typedef struct {
    int     width;          /* native display width in pixels */
    int     height;         /* native display height in pixels */
    float   refresh_rate;   /* e.g. 60.0, 120.0 */
    float   scale;          /* logical scale factor (1.0 initially) */
    int     orientation;    /* 0 = landscape, 1 = portrait */
    int     hdr_supported;  /* 1 if HDR is available (post-MVP) */
} PlayOSDisplayInfo;

int playos_display_get_info(PlayOSDisplayInfo *info);

/* Request v-sync preference. PlayOS may or may not honor it.
   0 = disabled, 1 = enabled (default). Returns 0 if accepted. */
int playos_display_set_vsync(int enabled);
```

---

### 5.5 `playos_storage.h`

Per-game storage paths and helpers. Paths are set from the environment at launch.

```c
/* All returned paths are valid for the process lifetime. Returns NULL if unavailable. */
const char *playos_storage_get_install_path(void);  /* /data/games/<game-id>  read-only */
const char *playos_storage_get_saves_path(void);    /* /data/saves/<game-id>  read-write */
const char *playos_storage_get_cache_path(void);    /* /data/cache/<game-id>  read-write */

/* Shell-only: returns the games root path. Not available to game processes. */
const char *playos_storage_get_games_path(void);

/* Free space on the data partition in bytes. */
int64_t playos_storage_free_bytes(void);

/* Atomically replace dst_path with the content in src_path (rename-based).
   Returns 0 on success. */
int playos_storage_atomic_replace(const char *src_path, const char *dst_path);

/* Write data to a temporary file, then atomically rename it to path.
   Returns 0 on success. */
int playos_storage_atomic_write(const char *path, const void *data, size_t len);
```

**Isolation guarantee:** Each game receives paths scoped to its `game_id`. A game process cannot construct or access another game's save path through this API. Landlock enforcement (Sprint 12) backs this up at the OS level.

---

### 5.6 `playos_audio.h`

System audio state. Games control their own streams through Raylib; this API exposes system-level info.

```c
typedef struct {
    int     sample_rate;      /* e.g. 44100 or 48000 */
    int     channels;         /* 1 or 2 */
    int     bits_per_sample;  /* 16 */
    float   master_volume;    /* 0.0 – 1.0, system-wide */
    int     muted;            /* 1 if system is muted */
} PlayOSAudioInfo;

int playos_audio_get_info(PlayOSAudioInfo *info);

/* Request system volume change. Only honored when the game is foreground.
   Returns 0 if accepted, -1 if denied. */
int playos_audio_set_master_volume(float volume);
int playos_audio_set_muted(int muted);
```

---

### 5.7 `playos_power.h`

Battery, thermal, and performance profiles.

```c
typedef enum {
    PLAYOS_POWER_STATE_ON_BATTERY,
    PLAYOS_POWER_STATE_CHARGING,
    PLAYOS_POWER_STATE_CHARGED,
    PLAYOS_POWER_STATE_UNKNOWN
} PlayOSPowerState;

typedef enum {
    PLAYOS_THERMAL_NORMAL,    /* < 75°C */
    PLAYOS_THERMAL_WARM,      /* 75–85°C */
    PLAYOS_THERMAL_HOT,       /* 85–95°C — system may reduce performance */
    PLAYOS_THERMAL_CRITICAL   /* ≥ 95°C — system will shut down */
} PlayOSThermalState;

typedef enum {
    PLAYOS_PERF_BALANCED,     /* system-managed (default) */
    PLAYOS_PERF_POWER_SAVE,   /* low TDP, extended battery */
    PLAYOS_PERF_PERFORMANCE   /* high TDP, best GPU/CPU */
} PlayOSPerfProfile;

typedef struct {
    PlayOSPowerState   power_state;
    int                battery_percent;   /* 0–100; -1 if unknown */
    int                minutes_remaining; /* -1 if unknown or charging */
    PlayOSThermalState thermal_state;
    int                cpu_temp_c;
    int                gpu_temp_c;
    PlayOSPerfProfile  active_profile;
} PlayOSPowerInfo;

int playos_power_get_info(PlayOSPowerInfo *info);

/* Request a performance profile. PlayOS may deny or override based on thermal state.
   Returns 0 if accepted, -1 if denied. */
int playos_power_request_profile(PlayOSPerfProfile profile);
```

---

### 5.8 `playos_logging.h`

Structured logging routed to the PlayOS log system.

```c
typedef enum {
    PLAYOS_LOG_DEBUG = 0,
    PLAYOS_LOG_INFO  = 1,
    PLAYOS_LOG_WARN  = 2,
    PLAYOS_LOG_ERROR = 3
} PlayOSLogLevel;

/* Log a structured message.
   tag: short category string, e.g. "audio", "render", "save"
   fmt: printf-style format string */
void playos_log(PlayOSLogLevel level, const char *tag, const char *fmt, ...);

/* Convenience macros */
#define PLAYOS_LOG_D(tag, ...) playos_log(PLAYOS_LOG_DEBUG, tag, __VA_ARGS__)
#define PLAYOS_LOG_I(tag, ...) playos_log(PLAYOS_LOG_INFO,  tag, __VA_ARGS__)
#define PLAYOS_LOG_W(tag, ...) playos_log(PLAYOS_LOG_WARN,  tag, __VA_ARGS__)
#define PLAYOS_LOG_E(tag, ...) playos_log(PLAYOS_LOG_ERROR, tag, __VA_ARGS__)

/* Mark a crash point before calling abort() or raising a fatal signal. */
void playos_log_crash_marker(const char *reason);
```

Log output is written to `/data/log/<game-id>/session-<timestamp>.log` and to the kernel ring buffer (for development builds).

---

## 6. Backend Architecture

`libplayos` uses an internal backend interface to abstract platform details. The backend is selected at runtime via the `PLAYOS_BACKEND` environment variable or auto-detection.

```
playos_input_get_controller_state()
    │
    ▼
PlayOSInputBackend.get_controller_state()
    │
    ├── "evdev"    → reads from /dev/input/event* via Linux evdev
    └── "stub"     → returns zeroed state (testing/CI)
```

The backend is an internal detail. Games always call the public API; the backend is never exposed.

---

## 7. C++ and Engine Wrappers

`playos-platform-api` may provide optional C++ wrappers and engine adapters:

- `include/playos/playos.hpp` — C++ RAII wrappers and enum class aliases
- `src/backends/rcore_playos.c` — Raylib platform backend
- Future: Godot, SDL2 adapter headers

These wrappers **must not replace the C ABI** as the source of truth. They are convenience layers built on top of it.

---

## 8. What the API Must Never Expose

| Forbidden | Reason |
|---|---|
| DRM file descriptors or device paths | Direct display access belongs to `playos-compositor` |
| Wayland display or socket handle | Wayland connection is managed by the Raylib backend |
| `playos-runtime` IPC socket path or fd | Internal transport is private |
| Another game's storage paths | Isolation is a security guarantee |
| `playos-init` control IPC | Process control is a trusted-only path |
| Raw Linux input event codes | Logical mapping is the stable abstraction |
| Kernel module or firmware interfaces | Not a public game concern |

---

## 9. Change Process

1. **Additive changes** (new functions, new structs, new enum values): PR to `playos-platform-api`, reviewed against this spec.
2. **Potentially breaking changes**: RFC issue in `playos-spec`, reviewed by at least two contributors, resulting in an ADR.
3. **Breaking changes**: ADR required, major SONAME bump, migration guide, minimum 1-sprint notice before adoption in dependent repos.

*See [architecture.md §7.6](architecture.md) for the component overview.*
