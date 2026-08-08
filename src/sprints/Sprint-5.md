# Sprint 5 — Raylib-Powered PlayOS Shell

**Goal:** Build `playos-shell` as a hardware-accelerated, controller-first Raylib Wayland client that runs persistently under `playos-compositor` and consumes the public `libplayos` API surface needed for shell UX.

**Primary Outcome:** The ROG Ally boots into a visible shell UI that shows a stub game library, responds to controller navigation, and remains alive as the persistent PlayOS foreground experience.

**Prerequisites:** Sprint 4 complete — the compositor owns the real display on the Ally. Sprint 3 complete — the public input ABI is finalized (bit positions resolved), evdev backend is implemented, and the `playos-platform-api` headers are in place with stub implementations.

---

## Why This Sprint Exists

Sprint 5 is the first real user-facing PlayOS sprint. Everything before it proves build, runtime, hardware, and graphics foundations. This sprint proves that those foundations are sufficient to host the persistent console shell that defines the product experience.

---

## Start Condition Checklist

- Sprint 4 native DRM/KMS path works on the Ally.
- `playos-compositor` can present a diagnostic client on real hardware.
- `playos-platform-api` already has all 8 public headers declared, the Sprint 3 input contract is finalized (bit positions fixed, evdev backend implemented), and stubs exist for system/storage/lifecycle/logging.
- The Sprint 3 critical review finding (input header bit position mismatch) has been resolved — current `playos_input.h` matches the spec.
- The shell can rely on a working Wayland session and hardware-accelerated rendering path.

---

## Decisions Locked for This Sprint

- **Language:** C99
- **UI framework:** Raylib
- **Windowing model:** one fullscreen shell surface only
- **Input model:** controller-first; mouse and keyboard are developer-only aids, not product requirements
- **Backend ownership:** the custom Raylib PlayOS backend is maintained by `playos-shell`; `playos-platform-api` provides the public API consumed by the shell. **Exception for input:** the shell needs SYSTEM/QUICK_MENU button access, which `libplayos` input API strips (those buttons are reserved, never delivered to game processes). The shell reads controller input directly through the evdev backend provided by `playos-platform-api` or through a future trusted compositor protocol.
- **Library content source for this sprint:** stub manifests in `/data/games/` (retrieved via `playos_storage_get_games_path()`) 
- **Launch behaviour:** the shell may issue a stub launch request or present a placeholder transition, but a full playable game lifecycle is deferred to Sprint 7
- **Battery UI:** no real power API dependency yet; use placeholder or omit battery if the real power contract is not available

---

## Scope

### In Scope

- custom Raylib backend for the PlayOS Wayland environment
- shell application structure and screen flow
- public API groups needed by the shell this sprint
- controller navigation
- stub game discovery from `/data/games/`
- Buildroot packaging for Raylib and the shell
- physical-hardware visual validation on the Ally

### Explicitly Out of Scope

- full game launch/resume/background lifecycle
- overlay UI
- real save-management UX
- network/store/account features
- real battery/power management policy
- installer/update flows

---

## Required Repository Changes

| Repo | Required work |
|---|---|
| `playos-shell` | Raylib shell app, custom PlayOS backend integration, screens, controller navigation |
| `playos-platform-api` | Headers already exist. Remaining work: add `playos_storage_get_games_path()`, implement the 4 stubs (system/storage/lifecycle/logging) with real backends, hand off to refdistro for libplayos packaging |
| `playos-refdistro` | Raylib packaging, `playos-shell` packaging, stub content under `/data/games/` |
| `playos-spec` | shell UX conventions and any clarified shell/runtime contract notes |

---

## Expected Files and Directories

### `playos-shell`

```text
CMakeLists.txt
include/
└── shell.h

src/
├── main.c
├── shell.c
├── navigation.c
├── library_model.c
├── launcher_stub.c
├── lifecycle.c
├── input.c
├── ui/
│   ├── screen_library.c
│   ├── screen_game_detail.c
│   └── status_bar.c
└── platforms/
    └── rcore_playos.c

assets/
├── fonts/
├── icons/
└── audio/
```

### `playos-platform-api`

All headers and source stubs already exist on disk. Sprint 5 adds one new function (`playos_storage_get_games_path`) and implements the stubs:

```text
include/playos/
├── playos_storage.h         ← EXTEND: add get_games_path() declaration

src/
├── playos_system.c          ← IMPLEMENT: real sysinfo from /proc + sysfs
├── playos_storage.c         ← IMPLEMENT: real paths + get_games_path()
├── playos_lifecycle.c       ← IMPLEMENT: real event-fd from IPC
└── playos_logging.c         ← IMPLEMENT: real structured logging to file
```

### `playos-refdistro`

```text
br2-external/package/
├── raylib/
└── playos-shell/

br2-external/board/common/rootfs-overlay/
└── data/games/
    ├── com.playos.demo1/manifest.json
    ├── com.playos.demo2/manifest.json
    └── com.playos.demo3/manifest.json
```

---

## Agent Task Breakdown

### Task Status Grid

Update the **Status** column as work progresses: `not started` → `in progress` → `blocked` or `done`.

| Task ID | Task | Primary repo | Status | Notes / evidence |
|---|---|---|---|---|
| S5-T1 | Finalise the shell-facing public API surface | `playos-platform-api` | in progress | Headers exist. Stubs exist. Needs: `get_games_path()`, real impls. |
| S5-T2 | Add the custom Raylib PlayOS backend | `playos-shell` | not started | |
| S5-T3 | Bootstrap the shell application structure | `playos-shell` | not started | |
| S5-T4 | Implement library data loading from stub manifests | `playos-shell`, `playos-refdistro` | not started | |
| S5-T5 | Implement controller-first navigation and focus rules | `playos-shell` | not started | |
| S5-T6 | Build the library, detail, and status-bar UI | `playos-shell` | not started | |
| S5-T7 | Add shell lifecycle handling and persistent process behavior | `playos-shell`, `playos-platform-api` | not started | |
| S5-T8 | Integrate Raylib and shell packaging into Buildroot | `playos-refdistro` | not started | |
| S5-T9 | Add validation, stub content, and runtime evidence capture | `playos-shell`, `playos-refdistro` | not started | |

### S5-T1 — Finalise the shell-facing public API surface

> **Note:** All 4 header files for this task already exist with full declarations. The evdev input backend is already implemented. Source files exist as stubs (return NULL/0/-1). This task is about **extending with one missing function** and **implementing the stubs** for Sprint 5's minimum needs.

**Update `playos_storage.h` — add the one missing function:**

```c
/**
 * Read-only path to the game library directory.
 * e.g. /data/games/
 *
 * @return  Null-terminated path string.
 */
const char *playos_storage_get_games_path(void);
```

This function is needed because the shell is NOT a game — it doesn't have `PLAYOS_GAME_ID` set. The existing storage API assumes the caller is a game process. The shell needs `get_games_path()` to discover installed game manifests.

**Implement the 4 stubs for Sprint 5:**

| Function | Implementation target |
|---|---|
| `playos_system_*()` | Read from `/proc/cpuinfo`, `/proc/meminfo`, `/sys/class/drm/`; hardcode device model |
| `playos_storage_*()` | Base paths from `/data/games/`, `/data/saves/`, `/data/cache/`; `free_bytes` via `statvfs` |
| `playos_lifecycle_*()` | Create private `eventfd`; wire to IPC from playos-init when available, poll-only stub for now |
| `playos_log_*()` | Write to file under `/run/playos/log/` with timestamp; `crash_marker` via `sync()` + `write()` |

The type names used in the actual headers differ from the draft in this spec — use the actual types already declared:
- `PlayOSLifecycleEvent` (not `playos_lifecycle_event_t`)
- `PlayOSLogLevel` (not `playos_log_level_t`)
- `playos_storage_get_saves_path(void)` and `playos_storage_get_cache_path(void)` — no `game_id` parameter (game isolation is via `PLAYOS_GAME_ID` env var)

**Done when:** the shell can compile and link against `libplayos.so` for system, storage, lifecycle (poll-only), and logging needs.

### S5-T2 — Add the custom Raylib PlayOS backend

Create `src/platforms/rcore_playos.c` in `playos-shell`.

The backend must:

- create a Wayland fullscreen surface
- create an EGL context
- integrate frame pacing with Wayland frame callbacks
- feed controller state from `playos_input_get_controller_state()`
- cooperate with the PlayOS compositor environment instead of desktop assumptions

Desktop-only features must be disabled or become no-ops:

- window decorations
- free resize
- drag and drop
- clipboard
- multi-window support

**Done when:** a simple Raylib frame can be drawn through the custom backend on the Ally.

### S5-T3 — Bootstrap the shell application structure

- Add a central `struct playos_shell`.
- Add screen/state enums.
- Add a main loop that separates:
  1. event/input polling
  2. state update
  3. draw

- Ensure startup and shutdown paths are explicit.

**Done when:** the shell launches, draws a frame, and exits cleanly under developer control.

### S5-T4 — Implement library data loading from stub manifests

- Read stub game entries from `/data/games/`.
- Parse minimal `manifest.json` data for:
  - game id
  - display name
  - version
  - optional description
  - optional icon path

- Use placeholder visuals when optional data is absent.

**Done when:** the shell can load and display three deterministic stub entries.

### S5-T5 — Implement controller-first navigation and focus rules

- D-pad moves selection
- `A` confirms/selects
- `B` returns/back
- focus must always remain visible and unambiguous
- no mouse is required for normal operation

> **⚠️ Trusted vs untrusted input:** The shell is a **trusted** system component — it needs the SYSTEM (Xbox Guide) and QUICK_MENU (Ally Armoury Crate) buttons. The public `playos_input_get_controller_state()` function strips these reserved buttons before returning (since games must never see them). The shell must either:
>   1. Link the evdev backend directly (bypassing `libplayos` for input), **or**
>   2. Consume input through a compositor protocol (e.g., the future `session_manager` protocol in `playos-v1.xml`)
>
> For Sprint 5, option 1 (direct evdev) is the pragmatic path — the shell already runs on the same system as the evdev input backend and can access `/dev/input/` directly. Long term, a privileged input protocol in the compositor is preferred.

Recommended first screen flow:

```text
Library -> Game Detail -> Library
```

**Done when:** a user can navigate into and out of the detail screen with controller input only.

### S5-T6 — Build the library, detail, and status-bar UI

Required UI surfaces for this sprint:

1. **Library screen**
   - scrollable grid or list of installed games
   - visible focus state
   - placeholder icon support

2. **Game detail screen**
   - game name
   - description
   - version
   - launch/select affordance (may be stubbed)

3. **Status bar**
   - PlayOS version
   - clock
   - optional placeholder battery field if clearly marked as non-final

**Done when:** the shell renders a coherent controller-first UI rather than a single diagnostic frame.

### S5-T7 — Add shell lifecycle handling and persistent process behavior

- Poll lifecycle events through the public API if available.
- At minimum, define and handle:
  - foreground
  - background
  - terminate

- When backgrounded, reduce or skip rendering work.
- When foregrounded, resume normal rendering.
- On terminate, exit cleanly.
- The shell must be written as a persistent supervised process, not a one-shot demo app.

**Done when:** lifecycle state changes affect rendering behaviour predictably and the shell remains compatible with supervision.

### S5-T8 — Integrate Raylib and shell packaging into Buildroot

- Add `playos-shell` package metadata.
- Add Raylib with the custom PlayOS backend. **Strategy:** Create a vendored Raylib source in `playos-shell` rather than patching upstream — the `rcore_playos.c` backend replaces core platform code (`rcore_desktop.c` / `rcore_desktop_glfw.c`) and is tightly coupled to the PlayOS compositor. A Buildroot package references this vendored source.
- Ensure the shell depends on `libplayos`.
- Make the image start the real shell after compositor readiness.

**Done when:** the Ally image boots into the shell automatically.

### S5-T9 — Add validation, stub content, and runtime evidence capture

- Install three stub manifests under `/data/games/`.
- Add runtime logging on startup, shutdown, navigation, and lifecycle transitions.
- Validate shell startup on the Ally.
- Validate non-crashing startup in the QEMU/dev path, even if visual output is limited there.

**Done when:** the sprint has demonstrable content and logs proving the shell path works.

---

## Implementation Guidance

### Shell UX rules

- Always favour immediate controller clarity over dense visuals.
- Do not introduce nested menus beyond the single detail screen in this sprint.
- Keep animation and visual effects light until the shell baseline is stable.

### Manifest format

For Sprint 5, keep the manifest intentionally small. A minimal JSON shape is enough:

```json
{
  "id": "com.playos.demo1",
  "name": "Demo One",
  "version": "0.1.0",
  "description": "Stub entry used for shell bring-up."
}
```

**Note:** The shell discovers manifests from `/data/games/` using `playos_storage_get_games_path()`. This function is added to `playos-platform-api` in S5-T1. The shell is not a game process, so it does not have `PLAYOS_GAME_ID` set and must use this shell-specific discovery API rather than the per-game path functions.

### Lifecycle scope

This sprint may consume placeholder lifecycle events needed to keep the shell architecture clean, but it must not depend on the full console game/background/resume model from Sprint 7 being complete.

### Logging

At minimum log:

- shell startup
- backend initialization success/failure
- manifest load count
- selection changes
- screen transitions
- lifecycle transitions

---

## Verification and Evidence

| Evidence | How it is produced |
|---|---|
| UI boot proof | photo/video or direct observation of the shell on the Ally |
| Data-loading proof | shell log showing three manifests loaded |
| Navigation proof | log or captured session showing controller-driven focus changes |
| API proof | successful compile/link against public `libplayos` headers |
| Persistence proof | shell remains alive under supervision during idle runtime |
| Regression proof | shell starts in the non-device path without crashing |

---

## Acceptance Criteria

- [ ] the Ally boots into the shell UI automatically
- [ ] the shell is rendered through the custom Raylib PlayOS backend
- [ ] the shell links against documented public `libplayos` APIs only
- [ ] three stub game entries are loaded from `/data/games/`
- [ ] controller-only navigation works on the library and detail screens
- [ ] `A` enters the detail screen and `B` returns to the library
- [ ] the shell logs startup, navigation, and lifecycle events
- [ ] the shell remains alive under supervision during an idle run
- [ ] Buildroot packaging integrates Raylib and `playos-shell`
- [ ] the non-device startup path remains usable for developer iteration

---

## Handoff to Sprint 6

Sprint 6 may assume:

- the shell exists as the persistent UI process
- the Raylib PlayOS backend is real
- shell navigation and rendering fundamentals are stable
- a basic content-loading path from `/data/games/` already exists

Sprint 6 should deepen storage and real discovery behaviour rather than rebuilding shell fundamentals.

---

## Exit Gate

The ROG Ally boots directly into a persistent Raylib shell that renders through the PlayOS backend, loads stub game entries, and supports controller-first navigation using the public `libplayos` API surface.

*Previous: [Sprint 4](Sprint-4.md) | Next: [Sprint 6](Sprint-6.md)*
