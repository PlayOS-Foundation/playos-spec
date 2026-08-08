# Sprint 1 — `playos-init` and Minimal Boot Supervision

**Goal:** Replace the BusyBox `/init` stub with a real `playos-init` written in C99 that acts as PID 1, mounts the system, supervises the compositor, and exposes the first version of the trusted control IPC.

**Primary Outcome:** The system boots to a supervised state. `playos-init` is PID 1, the expected virtual filesystems are mounted, the data partition is discovered and mounted, and a trusted client can connect to `/run/playos/control.sock` and exchange versioned status and lifecycle messages.

**Prerequisites:** Sprint 0 complete — the Buildroot factory boots a PlayOS EFI image in QEMU/OVMF and the six repositories already exist locally.

---

## Why This Sprint Exists

Sprint 1 establishes the first real runtime contract for the platform:

1. A deterministic PID 1 implementation exists and owns system bring-up.
2. Process supervision exists before graphics, shell, or games become real.
3. The private trusted control channel exists before any higher-level component depends on it.

If this sprint is weak, every later sprint inherits undefined startup and lifecycle behaviour.

---

## Start Condition Checklist

Do not start implementation until all of the following are true:

- `playos-refdistro` can still boot the Sprint 0 QEMU image.
- The boot artifact still uses the Buildroot `br2-external` tree from `playos-refdistro`.
- `playos-runtime` exists and is available for shared IPC headers/helpers.
- No later sprint code is assumed to exist yet. `playos-compositor` may still be a stub process.

---

## Decisions Locked for This Sprint

These choices are intentionally fixed so an implementation agent does not need to guess:

- **Language:** use **C99** for `playos-init` in this sprint.
- **Location:** implement the source under `playos-refdistro\src\playos-init\`.
- **IPC transport:** Unix domain socket only.
- **IPC framing:** use the versioned framing defined in `..\runtime-ipc.md` (`PLOS` magic + length + JSON body).
- **Trusted access policy:** only processes in group `playos-trusted` may connect.
- **Recovery behaviour:** if the data partition is missing or the compositor exceeds restart limits, log a clear diagnostic and halt. Do not invent an interactive recovery UI yet.

---

## Scope

### In Scope

- PID 1 implementation
- Virtual filesystem mounting
- Data partition discovery and mount
- Minimal process supervision
- Trusted control IPC server
- Stub game launch / termination flow
- Buildroot packaging and boot integration
- Host and QEMU test coverage

### Explicitly Out of Scope

- Real graphics or Wayland logic
- Shell UI
- Real game metadata or library scanning
- Overlay, suspend, audio, or installer work
- Disk formatting or automatic repair

---

## Required Repository Changes

| Repo | Required work |
|---|---|
| `playos-refdistro` | Add the real `playos-init` source tree, package metadata, boot integration, and QEMU tests |
| `playos-runtime` | Add shared IPC framing/types/helpers used by PID 1 and test clients |
| `playos-spec` | Update specs if implementation forces a protocol clarification or ADR |

---

## Expected Files and Directories

The sprint is not complete unless these paths exist or are intentionally replaced with equivalent documented paths.

### `playos-refdistro`

```text
src/playos-init/
├── CMakeLists.txt
├── include/playos-init/
│   ├── init.h
│   ├── mount.h
│   ├── supervisor.h
│   └── recovery.h
├── src/
│   ├── main.c
│   ├── mount.c
│   ├── logging.c
│   ├── supervisor.c
│   ├── child_process.c
│   ├── recovery.c
│   └── shutdown.c
└── tests/
    ├── host/
    └── qemu/

br2-external/package/playos-init/
├── Config.in
└── playos-init.mk
```

### `playos-runtime`

```text
include/playos-runtime/
└── ipc.h

src/
├── ipc_framing.c
├── ipc_server.c
├── ipc_client.c
└── lifecycle_fd.c
```

---

## Agent Task Breakdown

Every task below is meant to be independently checkable in code review or testing.

### Task Status Grid

Update the **Status** column as work progresses: `not started` → `in progress` → `blocked` or `done`.

| Task ID | Task | Primary repo | Status | Notes / evidence |
|---|---|---|---|---|
| S1-T1 | Bootstrap the `playos-init` source tree | `playos-refdistro` | done | CMakeLists.txt, init.h, init.c, test_init_state.c |
| S1-T2 | Implement mandatory PID 1 boot responsibilities | `playos-refdistro` | done | mount.c, logging.c, shutdown.c, child_process.c |
| S1-T3 | Discover and mount the data partition | `playos-refdistro` | done | mount.c scans PARTLABEL=playos-data, creates dirs |
| S1-T4 | Add minimal compositor supervision | `playos-refdistro` | done | supervisor.c with restart policy (5/10s limit) |
| S1-T5 | Implement the trusted control IPC server | `playos-refdistro`, `playos-runtime` | done | ipc_framing.c, ipc_server.c, ipc_client.c at /run/playos/control.sock |
| S1-T6 | Implement stub game lifecycle handling | `playos-refdistro`, `playos-runtime` | done | LaunchGame/TerminateGame via IPC, SIGCHLD reaper |
| S1-T7 | Integrate with Buildroot | `playos-refdistro` | done | cmake-package, installs as /init |
| S1-T8 | Add test coverage and evidence capture | `playos-refdistro`, `playos-runtime` | done | 4 QEMU integration tests all PASS, host tests PASS |

### S1-T1 — Bootstrap the `playos-init` source tree

- Create the `playos-refdistro\src\playos-init\` buildable project.
- Add a host-buildable `CMakeLists.txt`.
- Define `struct playos_init_state` as the central mutable state container.
- Ensure the code can compile on a Linux host without the full image build.

**Done when:** a host build produces a `playos-init` binary.

### S1-T2 — Implement mandatory PID 1 boot responsibilities

- Mount `/dev`, `/proc`, `/sys`, and `/run`.
- Create `/run/playos/` and `/run/playos/log/`.
- Initialize bounded logging at `/run/playos/log/init.log`.
- Write a boot marker file such as `/run/playos/boot-stage`.
- Reap child processes reliably (`SIGCHLD` handling or `waitpid` loop).

**Done when:** QEMU boot shows `playos-init` as PID 1 and all expected mounts exist.

### S1-T3 — Discover and mount the data partition

- Search by **documented label, UUID, or GPT partition GUID**.
- Mount the result at `/data`.
- Create first-boot directories if missing:

```text
/data/games
/data/saves
/data/system
/data/log
```

- If the partition is missing, log the reason and enter the provisioning halt path.

**Done when:** `/data` is mounted in QEMU and the expected top-level directories exist after first boot.

### S1-T4 — Add minimal compositor supervision

- Spawn a compositor placeholder process from configuration.
- Track compositor PID and exit reason.
- Restart on clean exit or crash up to a documented retry limit.
- After repeated failure, log the restart history and halt.

**Done when:** killing the compositor stub causes a restart; repeated failure enters recovery.

### S1-T5 — Implement the trusted control IPC server

- Listen on `/run/playos/control.sock`.
- Apply mode `0660`.
- Require membership in `playos-trusted`.
- Reject version mismatches explicitly.
- Implement the following initial message set using the runtime framing contract:

```text
Client -> Init
- QueryStatus
- LaunchGame
- TerminateGame
- Shutdown
- Reboot

Init -> Client
- StatusReport
- GameStarted
- GameExited
- GameCrashed
- Error
```

**Done when:** a host or QEMU test client can query status and receive a valid versioned response.

### S1-T6 — Implement stub game lifecycle handling

- Enforce exactly one foreground game process at a time.
- Validate that the launch target exists before exec.
- Set a minimal documented environment for child processes.
- Emit `GameStarted`, `GameExited`, and `GameCrashed` messages.
- Implement forced termination with timeout escalation.

**Done when:** a stub game process can be launched, queried, and terminated through IPC.

### S1-T7 — Integrate with Buildroot

- Add the Buildroot package for `playos-init`.
- Install the built binary as `/init`.
- Remove the BusyBox shell-script `/init` from the normal boot path.
- Keep BusyBox available only for developer and diagnostic workflows.

**Done when:** `make qemu-build && make qemu-run` boots through the real binary.

### S1-T8 — Add test coverage and evidence capture

- Host tests for message framing and parsing
- QEMU integration test for PID 1 identity, mounts, and status IPC
- QEMU integration test for compositor restart behaviour
- QEMU integration test for game launch / termination

**Done when:** the sprint has automated evidence, not only manual claims.

---

## IPC Contract for This Sprint

Use the same wire shape everywhere in this sprint.

```json
{
  "v": 1,
  "type": "QueryStatus"
}
```

Example response:

```json
{
  "v": 1,
  "type": "StatusReport",
  "compositor_pid": 42,
  "game_pid": null,
  "uptime_s": 17,
  "recovery_mode": false
}
```

**Rule:** do not invent a second ad hoc protocol for tests. Tests must exercise the same framing and version rules used by production code.

---

## Implementation Guidance

### Data partition discovery

The code must make the search strategy obvious in logs:

1. try documented GPT partition type GUID or partition label
2. if multiple candidates exist, log the ambiguity and fail safe
3. never auto-format
4. never silently fall back to the root filesystem

### Child process supervision

- `playos-init` must remain the parent for supervised children.
- Keep the restart policy in one place, e.g. `supervisor.c`.
- Store last exit reason and restart count in memory for status reporting.

### Logging

- Write human-readable timestamps if available.
- Keep log size bounded.
- Logging must not crash PID 1 if the log file cannot be opened; fall back to stderr/console early in boot and continue.

---

## Verification and Evidence

The implementation agent must leave the sprint with concrete evidence:

| Evidence | How it is produced |
|---|---|
| PID 1 proof | `/proc/1/comm` or `ps` from QEMU shell |
| Mount proof | `mount` or `/proc/mounts` output showing `/dev`, `/proc`, `/sys`, `/run`, `/data` |
| IPC proof | test client transcript for `QueryStatus` |
| Supervision proof | log showing compositor restart count increasing |
| Recovery proof | log showing retry limit exceeded and halt path entered |
| Game lifecycle proof | test client transcript for `LaunchGame` and `TerminateGame` |

---

## Acceptance Criteria

- [x] `playos-init` is PID 1 as verified by `/proc/1/comm` or `ps`
- [x] `/dev`, `/proc`, `/sys`, and `/run` are mounted by `playos-init`
- [x] the data partition is discovered, mounted at `/data`, and first-boot directories are created
- [x] `playos-init` supervises a compositor placeholder process and restarts it on exit
- [x] repeated compositor failure enters the documented recovery halt path
- [x] `/run/playos/control.sock` exists with mode `0660`
- [x] an authorized client can connect and receive `StatusReport`
- [x] an unauthorized client is rejected clearly
- [x] `LaunchGame` spawns a stub process and emits `GameStarted`
- [x] `TerminateGame` stops the stub process and emits `GameExited`
- [x] `Shutdown` performs an orderly halt path
- [x] zombie processes are reaped correctly
- [x] the Buildroot image boots through the real `/init` binary
- [x] host and QEMU tests cover framing, supervision, and lifecycle behaviour

---

## Handoff to Sprint 2

Sprint 2 may assume the following and must not re-invent them:

- `playos-init` can start and supervise a real compositor binary
- a trusted control socket already exists
- the system has a writable `/run/playos/` area for readiness markers and logs
- recovery-on-failure behaviour for the compositor already exists

Any missing readiness signal between PID 1 and the compositor should be added as an incremental Sprint 2 protocol extension, not a replacement.

---

## Exit Gate

`playos-init` runs as PID 1 in QEMU, mounts the system, supervises a compositor placeholder, mounts `/data`, and responds correctly to trusted versioned IPC commands.

*Previous: [Sprint 0](Sprint-0.md) | Next: [Sprint 2](Sprint-2.md)*
