# Sprint 1 — `playos-init` and Minimal Boot Supervision

**Goal:** Replace the BusyBox `/init` stub with a real `playos-init` written in C (or Rust) that acts as PID 1, mounts the system, and provides a versioned private control IPC skeleton.

**Primary Outcome:** The system boots to a supervised state. `playos-init` is alive as PID 1, virtual filesystems are mounted, the data partition is discovered and mounted, and a control IPC socket is accepting connections.

**Prerequisites:** Sprint 0 complete — UEFI boot pipeline and Buildroot factory working.

---

## Key Deliverables

### `playos-init` — PID 1 Process (`playos-refdistro/src/playos-init/`)

Implement as a small, deterministic C (or Rust) binary built under `playos-refdistro`.

**Boot responsibilities:**
- Mount `/dev` (devtmpfs), `/proc`, `/sys`, `/run` (tmpfs)
- Initialize logging to `/run/playos/log/init.log` (ring-buffer, bounded)
- Discover the PlayOS data partition by GUID, label, or UUID (no silent format)
- Mount the data partition at `/data` (ext4, read-write); enter provisioning mode if not found
- Set up `/data` top-level directories if absent (first boot only)
- Write a pidfile and a boot sequence marker

**Process supervision:**
- Start and supervise `playos-compositor` (stub for this sprint — a placeholder `sleep` binary is fine)
- Track compositor PID; restart it if it exits cleanly or crashes (up to a configurable retry limit)
- After repeated compositor failure, enter recovery mode (log + halt for now)
- Reap all zombie processes (act as a proper PID 1)
- Handle SIGTERM / SIGINT for graceful shutdown

**Game process lifecycle (skeleton):**
- Accept `LaunchGame` control IPC request (defined below)
- Validate: only one game at a time, executable must exist
- Spawn game with correct environment; supervise it
- On game exit or crash: record status, emit `GameExited` IPC event
- `TerminateGame` forcibly kills the game if it does not exit within timeout

**Shutdown / reboot:**
- Handle `Shutdown` and `Reboot` IPC commands
- Sync filesystems; kill processes in reverse order; call `reboot(2)` / `poweroff`

### `playos-runtime` — Private Control IPC Skeleton

Define the versioned IPC protocol that `playos-init` exposes to trusted clients.

**Transport:** Unix domain socket at `/run/playos/control.sock` (mode 0660, owned by playos-system group)

**Initial message types (binary or simple text framing — pick one and document it):**

```
Client → Init:
  LaunchGame      { game_id: string, manifest_path: string }
  TerminateGame   { game_id: string, force: bool }
  Shutdown        {}
  Reboot          {}
  QueryStatus     {}

Init → Client (events):
  GameStarted     { game_id: string, pid: u32 }
  GameExited      { game_id: string, pid: u32, exit_code: i32, signal: i32 }
  GameCrashed     { game_id: string, pid: u32 }
  StatusReport    { compositor_pid: u32, game_pid: u32 | null, uptime_s: u32 }
```

**Versioning:** Include a `protocol_version` field in every message. Reject mismatched versions with an explicit error. Define `PLAYOS_IPC_VERSION = 1`.

**Access control:** Only processes in the `playos-trusted` group may connect to the control socket. Document the group membership policy.

### Buildroot Integration
- Add `package/playos-init/` package to the `br2-external` tree
- `Config.in`, `playos-init.mk`, source reference
- Include `playos-init` binary in the initramfs as `/init`
- Remove the BusyBox `/init` script from the production path (keep BusyBox for development image only)

### Provisioning Mode
- If no data partition is found: log the situation, enter a minimal recovery/provisioning loop
- For this sprint: print a clear diagnostic message and halt — no interactive UI yet
- Document the expected partition GUID/label that `playos-init` searches for

---

## Acceptance Criteria

- [ ] `playos-init` is PID 1 as verified by `/proc/1/comm` or `ps`
- [ ] All virtual filesystems are mounted: `/dev`, `/proc`, `/sys`, `/run`
- [ ] Data partition is discovered, mounted at `/data`, and directories are created on first boot
- [ ] `playos-init` restarts the compositor stub (a `sleep` binary) when it exits
- [ ] After N restart failures, `playos-init` logs a failure and halts (recovery entry point)
- [ ] Control IPC socket exists at `/run/playos/control.sock`
- [ ] A test client can connect and receive a `StatusReport` response
- [ ] `LaunchGame` spawns a stub game process; `GameStarted` event is emitted
- [ ] `TerminateGame` kills the stub game; `GameExited` event is emitted
- [ ] `Shutdown` causes a clean system halt
- [ ] Zombie reaping: spawning and killing processes leaves no zombies
- [ ] Buildroot builds `playos-init` cleanly from source
- [ ] CI passes

---

## Repositories Primarily Involved

| Repo | Work |
|---|---|
| `playos-refdistro` | `playos-init` source, Buildroot package, initramfs integration |
| `playos-runtime` | IPC protocol definition, versioned message types, client stub library |
| `playos-spec` | ADR documenting IPC transport choice; update architecture notes |

---

## Testing Approach

- Unit tests for IPC message serialization/deserialization (host build)
- QEMU integration test: boot image, verify PID 1 identity, verify mounts, send IPC commands via test client
- QEMU test: kill the compositor stub; verify restart; kill it N times; verify halt
- No physical hardware required this sprint

---

## Exit Gate

`playos-init` runs as PID 1 in QEMU, mounts the system, supervises a stub compositor, and responds correctly to all defined IPC messages.

*Previous: [Sprint 0](Sprint-0.md) | Next: [Sprint 2](Sprint-2.md)*
