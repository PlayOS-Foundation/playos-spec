# `playos-init` Specification

> **Repository:** `playos-refdistro/src/playos-init/`  
> **Role:** PID 1, process supervisor, boot orchestrator  
> **Language:** C (or Rust)  
> **Cross-references:** [architecture.md](architecture.md) §6–7.1, [runtime-ipc.md](runtime-ipc.md), [Sprint-1.md](Sprint-1.md)

---

## Responsibilities

`playos-init` owns **process lifecycle and boot**. It does not own surfaces, focus, rendering, game logic, or network policy.

| Owns | Does NOT own |
|---|---|
| Boot and service lifecycle | UI or rendering |
| Virtual filesystem mounts | Input routing |
| Storage discovery and mount | Display configuration |
| Starting and supervising `playos-compositor` | Game-specific logic |
| Game launch validation | Network policy |
| Process spawning, monitoring, reaping | Package management |
| Lifecycle fd creation and event delivery | |
| Forced game pause/kill fallback | |
| Shutdown, reboot, factory reset, recovery | |

---

## Boot Sequence

```
Kernel starts /init (playos-init, PID 1)
    │
    ├── Mount /dev (devtmpfs), /proc, /sys, /run (tmpfs)
    ├── Open log sink: /run/playos/log/init.log (ring buffer, bounded)
    ├── Discover and validate data partition
    │     ├── Found → mount /data (ext4, rw)
    │     └── Not found → provisioning mode (halt with diagnostic)
    ├── First-boot: create /data directory tree
    ├── Create /run/playos/ directory tree
    ├── Bind control IPC socket: /run/playos/control.sock
    ├── Bind compositor control socket: /run/playos/compositor.sock
    │
    └── Start playos-compositor
          ├── Wait for compositor readiness signal (fd/pipe)
          └── On ready: compositor loop begins
```

---

## Process Supervision

`playos-init` acts as a proper PID 1 supervisor:

- **Zombie reaping:** Calls `waitpid(-1, WNOHANG)` in a loop on `SIGCHLD`
- **Compositor supervision:**
  - On compositor exit (any reason): record exit status, wait `COMPOSITOR_RESTART_DELAY_MS` (500ms), restart
  - After `COMPOSITOR_MAX_RESTARTS` (default: 3) within `COMPOSITOR_WINDOW_S` (default: 60s): enter recovery mode
- **Game supervision:**
  - Track game PID and `game_id`
  - On game exit: emit `GameExited` IPC event; update internal state; unblock the shell
  - On crash: set `crashed=true` in `GameExited`
- **Overlay process:** Supervised same as compositor (restart on exit)

**Supervision table:**

| Process | Restart policy | Failure action |
|---|---|---|
| `playos-compositor` | Restart, up to N times | Recovery mode |
| `playos-shell` | Restart (shell is always alive) | Restart compositor session |
| `playos-overlay` | Restart | Log, continue |
| Active game | Never restart automatically | Emit `GameExited(crashed=true)` |

---

## Storage Discovery

`playos-init` searches for the data partition in order:

1. Partition with label `playos-data`
2. Partition with GUID `<defined in playos-spec/schemas/disk-layout.json>`
3. UUID from kernel command line: `playos.data_uuid=<uuid>`

**PlayOS must never silently format.** If the partition is not found:
- Log the search results (devices enumerated, labels found)
- Enter **provisioning mode**: display a diagnostic (installer handles the UI)
- Do not format, write, or modify any disk

---

## Game Launch Validation

Before spawning a game, `playos-init` validates:

| Check | Failure action |
|---|---|
| Only one game at a time | Return `LaunchGameError(already_running)` |
| Manifest file exists and is valid JSON | Return `LaunchGameError(invalid_manifest)` |
| `api_version` ≤ `PLAYOS_API_VERSION` | Return `LaunchGameError(unsupported_api_version)` |
| `architecture` matches running system | Return `LaunchGameError(invalid_manifest)` |
| Executable exists and is executable | Return `LaunchGameError(executable_not_found)` |
| Manifest `id` matches directory name | Return `LaunchGameError(invalid_manifest)` |

---

## Game Spawn Environment

`playos-init` prepares the following environment for the game process:

```
PLAYOS_GAME_ID=<game-id>
PLAYOS_INSTALL_PATH=/data/games/<game-id>
PLAYOS_SAVE_PATH=/data/saves/<game-id>
PLAYOS_CACHE_PATH=/data/cache/<game-id>
WAYLAND_DISPLAY=playos-0
PLAYOS_LIFECYCLE_FD=<fd>
PLAYOS_LAUNCH_TOKEN=<uuid4>
PLAYOS_API_VERSION=1
```

Before `execve()`, `playos-init`:
1. Sets `PLAYOS_GAME_ID`, paths, and lifecycle environment
2. Applies `PR_SET_NO_NEW_PRIVS = 1`
3. Drops all capabilities
4. Applies seccomp filter (Sprint 12)
5. Applies Landlock rules (Sprint 12)
6. Drops `CAP_SETUID` / `CAP_SETGID`
7. `execve()` the game executable

---

## Cooperative vs Forced Termination

On `TerminateGame`:

```
playos-init sends SIGTERM to game
    │
    ├── Game exits within GAME_EXIT_TIMEOUT_MS (default: 2000ms) → clean exit
    │
    └── Timeout expires
          │
          playos-init sends SIGKILL
              │
              Game process is reaped; GameExited(crashed=false, force_killed=true) emitted
```

Non-cooperative backgrounding fallback (Sprint 7):
```
PLAYOS_LIFECYCLE_BACKGROUND delivered to game via lifecycle fd
    │
    ├── Game reduces CPU within GAME_PAUSE_TIMEOUT_MS (default: 500ms) → OK
    │
    └── Timeout: playos-init sends SIGSTOP to game process
```

`SIGCONT` is sent when the compositor transitions back to `GAME_FOREGROUND`.

The compositor **requests** SIGSTOP/SIGCONT via the compositor control socket; it does not send signals itself.

---

## Shutdown and Reboot

On `Shutdown` or `Reboot` IPC:

1. Deliver `PLAYOS_LIFECYCLE_TERMINATE` to the active game (if any) via lifecycle fd
2. Wait up to 2 seconds for game to exit
3. Send SIGKILL to game if still alive
4. Send SIGTERM to compositor
5. Wait up to 2 seconds for compositor to exit
6. Sync all filesystems: `sync()`
7. Call `reboot(RB_POWER_OFF)` or `reboot(RB_AUTOBOOT)`

---

## Recovery Mode

Entered when:
- The compositor restarts and fails more than `COMPOSITOR_MAX_RESTARTS` times
- `playos-init` receives a `RecoveryMode` IPC command
- Boot count exceeds A/B rollback limit and both slots are bad

Recovery mode:
1. Kill all non-init processes
2. Attempt to start a recovery UI (SimpleDRM or framebuffer, no AMDGPU required)
3. Show: log viewer, factory reset, rollback slot, reboot, shutdown options
4. No shell, no game launch, no compositor restart
