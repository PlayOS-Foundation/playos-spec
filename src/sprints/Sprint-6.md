# Sprint 6 — Persistent Storage and Game Discovery

**Goal:** Establish reliable persistent ext4 storage, a stable `/data` directory layout, safe `playos-platform-api` path conventions, and live game discovery in the shell from real manifest files.

**Primary Outcome:** Games installed in `/data/games/` are discovered, displayed in the shell, and their save and cache paths are correctly isolated per game. Data survives reboot.

**Prerequisites:** Sprint 5 complete — shell running on ROG Ally.

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
