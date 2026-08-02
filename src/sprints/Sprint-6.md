# Sprint 6 — Persistent Storage and Game Discovery

**Goal:** Establish reliable persistent ext4 storage, a stable `/data` directory layout, safe `playos-platform-api` path conventions, and live game discovery in the shell from real manifest files.

**Primary Outcome:** Games installed in `/data/games/` are discovered, displayed in the shell, and their save and cache paths are correctly isolated per game. Data survives reboot.

**Prerequisites:** Sprint 5 complete — Raylib shell running on the ROG Ally and consuming the public `libplayos` API.

---

## Why This Sprint Exists

Sprint 5 used stub data loaded from hardcoded entries. Sprint 6 replaces all of that with real persistent storage. Without this sprint, the shell is a demo; with it, the shell is a real game library. Every subsequent sprint depends on having reliable isolated per-game paths.

---

## Start Condition Checklist

- Sprint 5 shell boots and navigates on the Ally.
- `/data` partition discovery exists in `playos-init` (Sprint 1 skeleton) but the full provisioning and layout are still stubs.
- `playos_storage.h` was stubbed in Sprint 5 — this sprint makes it real.
- The QEMU image can be booted for host-side storage logic testing.

---

## Decisions Locked for This Sprint

- **Partition identification:** search order is GPT partition type GUID → label `playos-data` → UUID on kernel cmdline
- **Data directory schema:** the `/data/` layout defined here is final for MVP — do not change without an ADR
- **Manifest format:** v1 JSON schema defined here is the stable game metadata contract for MVP
- **`PLAYOS_GAME_ID` env var:** set by `playos-init` at game launch; storage API derives paths from it
- **FactoryReset scope this sprint:** `erase_cache` and `erase_config` only; saves and games are destructive and deferred to Sprint 10

---

## Scope

### In Scope

- `/data` partition mount, first-boot provisioning, directory tree creation
- `/data` directory schema (final MVP layout)
- Game manifest v1 schema
- Real `playos_storage.h` implementation
- Live game discovery in the shell
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
| `playos-refdistro` | Storage provisioning in `playos-init`, sample game packages, QEMU/Ally test content |
| `playos-platform-api` | Real `playos_storage.h` implementation |
| `playos-shell` | Live manifest-driven game discovery, icon loading, invalid manifest handling |
| `playos-runtime` | `FactoryReset` IPC command definition |
| `playos-spec` | Game manifest v1 schema file |

---

## Expected Files and Directories

### `playos-refdistro`

```text
src/playos-init/src/storage.c       # provisioning, mount, directory creation
br2-external/board/common/rootfs-overlay/data/games/
    com.playos.sample-triangle/
        manifest.json
        bin/triangle
    com.playos.sample-input/
        manifest.json
        bin/input-viewer
    com.playos.sample-audio/
        manifest.json
        bin/audio-stub
```

### `playos-platform-api`

```text
include/playos/playos_storage.h
src/playos_storage.c
```

### `playos-spec`

```text
schemas/game-manifest-v1.json
```

---

## Agent Task Breakdown

### Task Status Grid

Update the **Status** column as work progresses: `not started` → `in progress` → `blocked` or `done`.

| Task ID | Task | Primary repo | Status | Notes / evidence |
|---|---|---|---|---|
| S6-T1 | Implement `/data` partition provisioning in `playos-init` | `playos-refdistro` | not started | |
| S6-T2 | Define and create the final `/data` directory schema | `playos-refdistro` | not started | |
| S6-T3 | Define game manifest v1 schema | `playos-spec`, `playos-refdistro` | not started | |
| S6-T4 | Implement real `playos_storage.h` API | `playos-platform-api` | not started | |
| S6-T5 | Implement live game discovery in the shell | `playos-shell` | not started | |
| S6-T6 | Build and install three real sample games | `playos-refdistro` | not started | |
| S6-T7 | Add `FactoryReset` IPC command (cache/config scope) | `playos-runtime`, `playos-refdistro` | not started | |
| S6-T8 | Persistence and isolation validation | `playos-refdistro` | not started | |

### S6-T1 — Implement `/data` partition provisioning in `playos-init`

- Search for the data partition in order: GPT type GUID → label `playos-data` → UUID from kernel cmdline
- Mount at `/data` with ext4
- Validate `/data/.playos-storage-version` marker on every mount
- On first boot: create all top-level directories and write the version marker
- On missing partition: log clearly with all attempted identifiers and available block devices, then halt — never silently format

**Done when:** QEMU and Ally boot and show a correctly populated `/data` tree with the version marker present.

### S6-T2 — Define and create the final `/data` directory schema

```text
/data/
├── .playos-storage-version
├── games/<game-id>/{manifest.json, bin/, assets/, shaders/, licenses/}
├── saves/<game-id>/{profiles/, autosaves/, settings/}
├── cache/<game-id>/{shaders/, compiled-assets/, temporary/}
├── profiles/
├── resources/
├── downloads/
├── logs/
├── updates/
├── screenshots/
└── config/
```

**Done when:** all directories are created on first boot and the layout matches this schema exactly.

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

### S6-T4 — Implement real `playos_storage.h` API

```c
const char *playos_storage_get_games_path(void);
const char *playos_storage_get_saves_path(const char *game_id);
const char *playos_storage_get_cache_path(const char *game_id);
const char *playos_storage_get_config_path(void);
const char *playos_storage_get_logs_path(void);
int64_t     playos_storage_free_bytes(void);
int         playos_storage_atomic_write(const char *path, const void *data, size_t len);
```

- Derive per-game paths from `PLAYOS_GAME_ID` environment variable
- `playos_storage_atomic_write` writes to a temp file then renames into place
- Return `NULL` for unavailable paths; never construct paths to non-existent mounts

**Done when:** the shell and sample games can call the API and receive correct isolated paths.

### S6-T5 — Implement live game discovery in the shell

- Call `playos_storage_get_games_path()`
- Enumerate subdirectories
- Read and parse `manifest.json` for each
- Validate against the manifest rules; skip and log invalid entries without crashing
- Load `assets/icon.png` if present; use placeholder if absent
- Sort results by name
- Replace the Sprint 5 stub list entirely

**Done when:** the shell shows discovered games dynamically from `/data/games/` on every startup.

### S6-T6 — Build and install three real sample games

- `com.playos.sample-triangle` — renders a Raylib triangle; proves GPU + Wayland path works
- `com.playos.sample-input` — displays live controller state
- `com.playos.sample-audio` — placeholder binary; audio will be wired in Sprint 8

Each must have a valid manifest and a compiled binary that at minimum starts without crashing.

**Done when:** all three appear in the shell library and can be selected in the detail screen.

### S6-T7 — Add `FactoryReset` IPC command (cache/config scope)

- Add `FactoryReset { erase_cache: bool, erase_config: bool }` to the IPC message set
- Implementation: unmount any sub-mounts, recursively delete selected directories, recreate them
- `erase_games` and `erase_saves` are defined in the schema but return an explicit "deferred to Sprint 10" error

**Done when:** `FactoryReset { erase_cache: true }` clears `/data/cache/` and the directory is recreated.

### S6-T8 — Persistence and isolation validation

- Write a file via `playos_storage_atomic_write` to a game's save path; reboot; read it back
- Verify two games have non-overlapping save paths
- Plant a broken manifest; verify shell skips it without crashing
- Verify `playos_storage_free_bytes()` returns a non-negative value

**Done when:** all persistence and isolation tests pass and are documented as evidence.

---

## Implementation Guidance

### Partition discovery order

Log every step: what was tried, what was found, why a candidate was accepted or rejected. This log is critical for debugging on new hardware.

### Atomic writes

The atomic write helper must be used for all persistent game data writes in the platform API. Raw `fwrite` to save paths is not acceptable.

### Manifest loading

Keep the manifest parser minimal and defensive. Unknown fields must be ignored, not rejected. Only fail on missing required fields or invalid values.

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

*Previous: [Sprint 5](Sprint-5.md) | Next: [Sprint 7](Sprint-7.md)*

---

## Key Deliverables

### `playos-init` — Storage Provisioning

**First-boot provisioning flow:**
1. Search for the data partition by: GUID → label `playos-data` → UUID from kernel cmdline
2. If found: mount at `/data`; verify `storage-version` marker
3. If not found: enter provisioning mode (see below)
4. After mount: ensure all required top-level directories exist:
   ```
   /data/{games,saves,profiles,resources,cache,downloads,logs,updates,screenshots,config}
   ```
5. Write or validate `/data/.playos-storage-version` (current format version)

**Provisioning mode (non-destructive first):**
- Log: device not found, expected GUID/label, available block devices
- For this sprint: halt with a diagnostic message (interactive provisioning UI is Sprint 10)
- PlayOS must never silently format an unknown disk

**Factory reset support (foundations):**
- Add an IPC command: `FactoryReset { erase_games: bool, erase_saves: bool, erase_cache: bool, erase_config: bool }`
- Implementation: unmount, selectively delete directories, re-create them, remount
- For this sprint: implement `erase_cache` and `erase_config` only; games and saves are destructive and deferred to Sprint 10

### `/data` Directory Layout (Final Schema)

```
/data/
├── .playos-storage-version         # format version marker
├── games/
│   └── <game-id>/
│       ├── manifest.json
│       ├── bin/
│       ├── assets/
│       ├── shaders/
│       └── licenses/
├── saves/
│   └── <game-id>/
│       ├── profiles/
│       ├── autosaves/
│       └── settings/
├── cache/
│   └── <game-id>/
│       ├── shaders/
│       ├── compiled-assets/
│       └── temporary/
├── profiles/
├── resources/
├── downloads/
├── logs/
├── updates/
├── screenshots/
└── config/
```

### Game Manifest Schema

Define the v1 manifest schema in `playos-spec/schemas/game-manifest-v1.json`:

```json
{
  "$schema": "...",
  "id": "com.example.game",
  "name": "Example Game",
  "version": "1.0.0",
  "executable": "bin/game",
  "api_version": 1,
  "graphics": "gles3",
  "architecture": "x86_64",
  "controllers": true,
  "network": false,
  "description": "Short game description.",
  "icon": "assets/icon.png"
}
```

**Validation rules:**
- `id` must match the directory name (reverse-domain format recommended)
- `executable` must exist relative to the game directory
- `api_version` must be ≤ the system's supported API version
- `architecture` must match the running system

### `playos-platform-api` — Storage API (Complete Implementation)

Implement `playos_storage.h` for real (it was stubbed in Sprint 5):

```c
/* All returned paths are valid for the lifetime of the process.
   Returns NULL if the path is unavailable. */
const char *playos_storage_get_games_path(void);
const char *playos_storage_get_saves_path(const char *game_id);
const char *playos_storage_get_cache_path(const char *game_id);
const char *playos_storage_get_config_path(void);
const char *playos_storage_get_logs_path(void);
int64_t     playos_storage_free_bytes(void);

/* Atomic file replacement helper:
   Writes to a temp file, then renames into place. */
int playos_storage_atomic_write(const char *path, const void *data, size_t len);
```

Storage paths are derived from the `PLAYOS_GAME_ID` environment variable (set by `playos-init` at launch).

### `playos-shell` — Live Game Discovery

Replace the hardcoded stub game list with real discovery:

1. Call `playos_storage_get_games_path()`
2. Enumerate subdirectories
3. For each: read and parse `manifest.json`
4. Validate the manifest; skip invalid entries with a log warning
5. Build the game list in memory; display in the shell UI
6. Load game icon from `assets/icon.png` if present; use a placeholder if not
7. Sort by game name

**Install test game packages:**
Add 3 real (minimal) games to the ROG Ally test image:
- `com.playos.sample-triangle` — a Raylib triangle (hello-world GPU test)
- `com.playos.sample-input` — displays controller input state in real time
- `com.playos.sample-audio` — plays a sine tone (Sprint 8 will make this work fully)

Each sample game must have a valid `manifest.json` and a compiled binary.

---

## Acceptance Criteria

- [ ] `/data` partition is mounted at boot; all directories exist
- [ ] `/data/.playos-storage-version` is written and validated on mount
- [ ] If the data partition is missing, `playos-init` logs clearly and halts — no silent format
- [ ] Shell discovers all games in `/data/games/` dynamically
- [ ] Games with invalid manifests are skipped (logged, not crashed)
- [ ] Game icons load if present; placeholder shown if absent
- [ ] `playos_storage_get_saves_path("com.playos.sample-triangle")` returns the correct isolated path
- [ ] A file written to a game's save path survives reboot (verified by reading it back after restart)
- [ ] Sample games appear in the shell UI after installation
- [ ] `FactoryReset { erase_cache: true }` clears `/data/cache/` and recreates it
- [ ] `playos_storage_free_bytes()` returns a non-negative value
- [ ] CI passes

---

## Repositories Primarily Involved

| Repo | Work |
|---|---|
| `playos-refdistro` | Data partition setup, first-boot provisioning in `playos-init`, sample game packages |
| `playos-platform-api` | Real `playos_storage.h` implementation, atomic write helper |
| `playos-shell` | Live game discovery, manifest parsing, icon loading |
| `playos-spec` | Game manifest schema v1, storage layout spec |
| `playos-runtime` | `FactoryReset` IPC command definition |

---

## Testing Approach

- Physical ROG Ally: boot, verify `/data` mount and directory tree
- Persistence test: write a file via `playos_storage_atomic_write`, reboot, read it back
- Discovery test: install a new game directory mid-session; verify it appears after shell restart
- Isolation test: verify two games have non-overlapping save paths
- Invalid manifest test: plant a broken `manifest.json`; verify shell skips it without crashing
- QEMU: storage provisioning on a virtual ext4 disk

---

## Exit Gate

Games installed in `/data/games/` are discovered and shown in the shell. Save and cache paths are correctly isolated per game. All data persists across reboots.

*Previous: [Sprint 5](Sprint-5.md) | Next: [Sprint 7](Sprint-7.md)*
