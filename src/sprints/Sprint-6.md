# Sprint 6 — Persistent Storage and Game Discovery

**Goal:** Establish reliable persistent ext4 storage, a stable `/data` directory layout, the real `playos-platform-api` storage path contract, and live game discovery in the shell from real manifest files.

**Primary Outcome:** Games installed in `/data/games/` are discovered, displayed in the shell, and their save and cache paths are correctly isolated per game. Data survives reboot.

**Status:** 🟡 Implemented on host — tasks S6-T1 through S6-T5 and S6-T7 are code-complete and building; S6-T6 ships three launchable samples but the GPU triangle rendering and live controller display are **deferred to a later sprint** (the samples currently prove only the launch + display/input query path); S6-T8 isolation is source-verified. Target runtime validation (QEMU/Ally boot, GPU/icon rendering, reboot persistence, live `FactoryReset` erase, cross-compiled samples) is still pending on hardware.

**Prerequisites:** Sprint 5.6 complete — repository boundaries are clean and every component's C source lives in its own repository.

---

## Why This Sprint Exists

Sprint 5 used stub data loaded from hardcoded entries, and storage was only stubbed behind the public API. Sprint 6 replaces the stub game list with real discovery over persistent storage. Without this sprint, the shell is a demo; with it, the shell is a real game library. Every subsequent sprint depends on reliable isolated per-game paths and on manifest-driven discovery.

Reality check as of Sprint 5.6: the storage API and a basic discovery scan already landed, but the full plan (manifest validation, icons, sorting, the complete `/data` schema, the version marker, sample games, and `FactoryReset`) is still outstanding.

---

## Start Condition Checklist

- [x] Sprint 5.6 complete; repository boundaries are clean (`playos-init` is its own repo, `playos-refdistro` has no C source).
- [x] `playos-init` already discovers and mounts a `playos-data` partition at `/data` (`src/mount.c`, `find_data_partition()`).
- [x] `playos-init` already creates a minimal first-boot directory set (`src/mount.c`, `playos_data_create_dirs()`).
- [x] The real `playos_storage.h` / `playos_storage.c` API shipped in Sprint 5 (`playos-platform-api`).
- [x] `playos-shell` already performs a basic `/data/games/` scan and reads `name` / `version` / `description` from `manifest.json` (`src/screen_library.c`).
- [x] `/data/.playos-storage-version` marker is written and validated by `playos-init` (`src/mount.c`).
- [x] Game manifest v1 schema exists (`playos-spec/schemas/game-manifest-v1.json`).
- [x] `playos-samples` contains three real sample games with manifests and built binaries.
- [x] `FactoryReset` handler exists in `playos-init` (JSON message specified in `playos-spec/src/runtime-ipc.md`).

---

## Decisions Locked for This Sprint

- **Partition identification (reconciled to implementation):** search order is label `playos-data` (via `/dev/disk/by-label/`, then a direct block-device scan, then `/proc/partitions`) → GPT partition type GUID (`PLAYOS_DATA_TYPE_GUID`) → UUID from the kernel cmdline `playos.data_uuid=`. When several `playos-data` partitions exist, removable media (USB) is preferred over internal NVMe; otherwise the last non-removable match wins. The earlier "GUID → label → UUID" wording is superseded by this actual order.
- **Data directory schema:** the `/data/` layout below is final for MVP. The shipped spelling is `/data/log` (singular); the spec docs have been reconciled to this spelling (only the non-editable `ideas.md` still shows the older plural).
- **Manifest format:** v1 JSON schema defined here is the stable game metadata contract for MVP.
- **`PLAYOS_GAME_ID` env var:** set by `playos-init` at game launch; the storage API derives per-game paths from it. Game ID is **not** a function argument.
- **FactoryReset scope this sprint:** `erase_cache` and `erase_config` only; `erase_games` and `erase_saves` are destructive and deferred to Sprint 10 (the schema may define them, but the handler returns an explicit "deferred" result).

---

## Scope

### In Scope

- `/data` partition mount (already present), first-boot provisioning, and the complete directory tree
- `/data` directory schema (final MVP layout)
- `/data/.playos-storage-version` marker (write on first boot, validate on mount)
- Game manifest v1 schema (`playos-spec/schemas/game-manifest-v1.json`)
- Real `playos_storage.h` implementation (already shipped — verify only)
- Complete live game discovery in the shell (validation, icons, sorting)
- Three real minimal sample games with valid manifests and compiled binaries
- `FactoryReset` IPC (cache and config only this sprint)

### Explicitly Out of Scope

- Game launch / lifecycle (Sprint 7)
- Audio (Sprint 8)
- Power / thermal (Sprint 9)
- Installer / disk formatting (Sprint 10)
- Full factory reset with save erasure (Sprint 10)

---

## Required Repository Changes

| Repo | Required work |
|---|---|
| `playos-init` | Extend `/data` provisioning: full directory schema, `.playos-storage-version` marker; add `FactoryReset` handler |
| `playos-platform-api` | Real `playos_storage` API (already implemented in Sprint 5 — verify, no new surface) |
| `playos-shell` | Complete manifest-driven discovery: full validation, icon loading, sorting, robust skip-on-invalid |
| `playos-samples` | Three real sample games with valid manifests and compiled binaries |
| `playos-refdistro` | Package/install the sample games into the rootfs overlay (no C source) |
| `playos-spec` | Game manifest v1 schema + this sprint doc |

> `FactoryReset` is a JSON IPC message already specified in `playos-spec/src/runtime-ipc.md`; `playos-init` owns both the message handling and the directory-erase logic (it is the IPC server owner). `playos-runtime` is not involved — its `protocols/playos-v1.xml` is the Wayland compositor protocol.

---

## Expected Files and Directories

### `playos-init`

```text
src/mount.c                    ← exists — extend playos_data_create_dirs() to the full schema + version marker
src/ipc_handler.c              ← add FactoryReset handler
```

### `playos-platform-api`

```text
include/playos/playos_storage.h   ← already real (Sprint 5)
src/playos_storage.c              ← already real (Sprint 5)
```

### `playos-shell`

```text
src/screen_library.c           ← exists — extend for validation, icon loading, sorting, skip-on-invalid
```

### `playos-samples`

```text
triangle/                      ← com.playos.sample-triangle
input-debug/                   ← com.playos.sample-input
audio-sine/                    ← com.playos.sample-audio
```

### `playos-refdistro`

```text
br2-external/board/common/rootfs-overlay/data/games/
    com.playos.sample-triangle/{manifest.json, bin/, assets/}
    com.playos.sample-input/{manifest.json, bin/, assets/}
    com.playos.sample-audio/{manifest.json, bin/, assets/}
```

### `playos-spec`

```text
schemas/game-manifest-v1.json
```

---

## Agent Task Breakdown

### Task Status Grid

| Task ID | Task | Primary repo | Status | Notes / evidence |
|---|---|---|---|---|
| S6-T1 | Implement `/data` partition provisioning | `playos-init` | done | Version marker write/validate + full provisioning added to `src/mount.c`; builds, `init_state` test passes |
| S6-T2 | Define and create the final `/data` directory schema | `playos-init` | done | `playos_data_create_dirs()` now creates the full 11-dir schema + marker |
| S6-T3 | Define game manifest v1 schema | `playos-spec` | done | `schemas/game-manifest-v1.json` created and JSON-valid |
| S6-T4 | Implement real `playos_storage` API | `playos-platform-api` | done (Sprint 5) | Verified canonical signature; no new surface |
| S6-T5 | Implement live game discovery in the shell | `playos-shell` | done (runtime pending) | Validation/icons/sort/skip-on-invalid added to `screen_library.c`; builds |
| S6-T6 | Build and install three real sample games | `playos-samples`, `playos-refdistro` | partial (rendering deferred) | Three samples build+run on host and install to `/data/games`; triangle renders nothing (display-query placeholder) and input-debug logs one snapshot — Raylib GPU rendering + live controller display deferred to a later sprint; target cross-compile unverified |
| S6-T7 | Add `FactoryReset` IPC command (cache/config scope) | `playos-init` | done (runtime pending) | JSON message per `runtime-ipc.md`; handler in `ipc_handler.c` + `ipc/ipc.h`; builds |
| S6-T8 | Persistence and isolation validation | `playos-refdistro` | partial (source-verified) | Isolation verified in source; reboot/QEMU persistence unverified |

---

### S6-T1 — Implement `/data` partition provisioning in `playos-init`

Current state: `playos-init/src/mount.c` `find_data_partition()` already implements the discovery order locked above (label first, removable-preferred, GPT GUID and cmdline UUID as fallbacks), mounts the partition at `/data` (ext4, with vfat/auto fallback), and writes a boot marker. The remaining work is the storage-version marker and the full provisioning flow.

- Validate `/data/.playos-storage-version` on every mount
- On first boot: create the full top-level directory set and write the version marker
- On missing partition: log clearly with all attempted identifiers and available block devices, then halt — never silently format

**Done when:** QEMU and Ally boot and show a correctly populated `/data` tree with the version marker present and validated.

---

### S6-T2 — Define and create the final `/data` directory schema

```text
/data/
├── .playos-storage-version
├── games/<game-id>/     {manifest.json, bin/, assets/, shaders/, licenses/}
├── saves/<game-id>/     {profiles/, autosaves/, settings/}
├── cache/<game-id>/     {shaders/, compiled-assets/, temporary/}
├── log/                 runtime logs (singular — matches shipped playos-init)
├── system/              system state (created by playos-init today)
├── profiles/
├── resources/
├── downloads/
├── updates/
├── screenshots/
└── config/
```

Current state: `playos_data_create_dirs()` creates only `/data/games`, `/data/saves`, `/data/system`, and `/data/log`. Sprint 6 must add `/data/cache`, `/data/profiles`, `/data/resources`, `/data/downloads`, `/data/updates`, `/data/screenshots`, `/data/config`, and the `.playos-storage-version` marker.

**Done when:** all directories are created on first boot and the layout matches this schema exactly.

---

### S6-T3 — Define game manifest v1 schema

Create `playos-spec/schemas/game-manifest-v1.json` (JSON Schema format) and document the required/optional fields:

```json
{
  "id": "com.example.game",
  "name": "Example Game",
  "version": "1.0.0",
  "executable": "bin/game",
  "api_version": 1,
  "graphics": "gles3",
  "architecture": "x86_64",
  "controllers": true,
  "network": false,
  "description": "Short description.",
  "icon": "assets/icon.png"
}
```

Validation rules:
- `id` must match the parent directory name
- `executable` must exist relative to the game directory
- `api_version` must be ≤ current system API version
- `architecture` must match the running system

**Done when:** the schema file exists and sample game manifests pass validation against it.

---

### S6-T4 — Implement real `playos_storage` API

This already shipped in Sprint 5. The canonical, verified signature is (no `game_id` parameters, no `config`/`logs` getters):

```c
const char *playos_storage_get_install_path(void);   /* /data/games/<game-id>   read-only */
const char *playos_storage_get_saves_path(void);     /* /data/saves/<game-id>   read-write */
const char *playos_storage_get_cache_path(void);     /* /data/cache/<game-id>   read-write */
const char *playos_storage_get_games_path(void);     /* /data/games             shell-only */
int64_t     playos_storage_free_bytes(void);         /* statvfs("/data"); -1 on error */
int         playos_storage_atomic_replace(const char *src_path, const char *dst_path);
int         playos_storage_atomic_write(const char *path, const void *data, size_t len);
```

- Per-game paths derive from `PLAYOS_GAME_ID`, set by `playos-init` at launch.
- `playos_storage_atomic_write` writes to a temp file then renames into place.
- Return `NULL` for unavailable paths; never construct paths to non-existent mounts.

**Done when:** verified present and matching the header (`include/playos/playos_storage.h`). No new API surface is needed this sprint.

---

### S6-T5 — Implement live game discovery in the shell

Current state: `playos-shell/src/screen_library.c` already calls `playos_storage_get_games_path()`, enumerates subdirectories with `opendir`, and parses `name` / `version` / `description` from `manifest.json` using a minimal `json_get_string` helper (fallback display name = directory name). The remaining work:

- Validate against the manifest rules (id/api_version/architecture/executable); skip and log invalid entries without crashing
- Load `assets/icon.png` if present; use a placeholder if absent
- Sort results by name
- Confirm the stub list is fully replaced (no hardcoded entries remain)

**Done when:** the shell shows discovered games dynamically from `/data/games/` on every startup, with icons, sorted, and robust to invalid manifests.

---

### S6-T6 — Build and install three real sample games

- `com.playos.sample-triangle` — placeholder that reports display info and exits; **Raylib GPU triangle rendering is deferred to a later sprint** (does not yet prove the GPU/Wayland rendering path)
- `com.playos.sample-input` — reports a single controller snapshot and exits; **live controller display is deferred** to a later sprint
- `com.playos.sample-audio` — placeholder binary; audio will be wired in Sprint 8

Each must have a valid manifest and a compiled binary that at minimum starts without crashing. These are built in `playos-samples` and packaged/installed into the rootfs overlay by `playos-refdistro` (no C source in refdistro).

**Done when:** all three appear in the shell library and can be selected in the detail screen. *(Met for the launch/query path; GPU triangle rendering and live controller display are explicitly deferred and tracked above.)*

---

### S6-T7 — Add `FactoryReset` IPC command (cache/config scope)

- `FactoryReset` is specified as a JSON IPC message in `playos-spec/src/runtime-ipc.md` with flags `erase_games`, `erase_saves`, `erase_cache`, `erase_config`, `erase_logs` (all default `false`).
- Implement the handler in `playos-init` (IPC server owner): reject when a game is running, recursively delete the selected directories, recreate them.
- `erase_cache` targets `/data/cache`; `erase_config` targets `/data/config`.
- `erase_games`, `erase_saves`, and `erase_logs` are reported as `"deferred"` in the response and acted on in Sprint 10.

**Done when:** `FactoryReset { erase_cache: true }` clears `/data/cache/` and the directory is recreated.

---

### S6-T8 — Persistence and isolation validation

- Write a file via `playos_storage_atomic_write` to a game's save path; reboot; read it back
- Verify two games have non-overlapping save paths
- Plant a broken manifest; verify the shell skips it without crashing
- Verify `playos_storage_free_bytes()` returns a non-negative value

**Done when:** all persistence and isolation tests pass and are documented as evidence.

---

## Implementation Guidance

### Partition discovery order

Log every step: what was tried, what was found, why a candidate was accepted or rejected. The existing `find_data_partition()` already logs the winning strategy — extend those logs, don't replace them.

### Atomic writes

The atomic write helper must be used for all persistent game data writes in the platform API. Raw `fwrite` to save paths is not acceptable.

### Manifest loading

Keep the manifest parser minimal and defensive. Unknown fields must be ignored, not rejected. Only fail on missing required fields or invalid values.

### Naming reconciliation

`/data/log` (singular) is the shipped spelling and is what this sprint locks. The spec docs (`architecture.md`, `platform-api.md`, `security-model.md`, and the sprint docs) have already been reconciled to `/data/log`; only the non-editable `ideas.md` retains the older plural.

---

## Verification and Evidence

| Evidence | How it is produced |
|---|---|
| Mount proof | `/proc/mounts` showing `/data` at boot |
| Directory proof | `ls /data` showing all expected subdirectories |
| Version marker proof | contents of `/data/.playos-storage-version` |
| Discovery proof | three sample games appear in shell library |
| Persistence proof | file written before reboot is readable after reboot |
| Isolation proof | two games' save paths are non-overlapping |
| Invalid manifest proof | shell log showing skipped invalid entry |

---

## Acceptance Criteria

- [ ] `/data` partition is mounted at boot; all directories exist
- [ ] `/data/.playos-storage-version` is written and validated
- [ ] Missing data partition causes a clear diagnostic halt with no silent format
- [ ] Shell discovers all games in `/data/games/` dynamically
- [ ] Invalid manifests are skipped and logged without crashing the shell
- [ ] Game icons load if present; placeholder shown otherwise
- [ ] Per-game save paths are correctly isolated
- [ ] A saved file survives reboot
- [ ] Three real sample games appear in the shell library
- [ ] `FactoryReset { erase_cache: true }` clears and recreates the cache directory
- [ ] `playos_storage_free_bytes()` returns a non-negative value

---

## Handoff to Sprint 7

Sprint 7 may assume:

- `/data` is reliably mounted with the final directory schema
- game manifests are validated on discovery
- `PLAYOS_GAME_ID` is an established pattern for per-game path isolation
- three sample games exist and are discoverable

Sprint 7 should focus on the launch flow, lifecycle, and overlay — not on re-implementing storage.

---

## Exit Gate

Games installed in `/data/games/` are discovered and shown in the shell. Save and cache paths are correctly isolated per game. All data persists across reboots.

*Previous: [Sprint 5.6](Sprint-5.6.md) | Next: [Sprint 7](Sprint-7.md)*
