# Sprint 12 — Security Hardening

**Goal:** Establish a hardened boundary between the public `playos-platform-api`, trusted `playos-runtime` control paths, and untrusted game processes. Games operate with minimal privileges. Remote debug services are removed from production builds. Secure Boot signing chain is defined.

**Primary Outcome:** A game process cannot access trusted IPC endpoints, cannot open DRM primary nodes, cannot write outside its own save/cache directories, and cannot synthesize reserved system input. Production builds ship without a shell or debug services.

**Prerequisites:** Sprint 11 complete — immutable images and A/B updates working.

---

## Key Deliverables

### Game Process Privilege Reduction

**Run games as an unprivileged identity:**
- `playos-init` spawns games as `playos-game` user (UID ~1000, no supplementary groups)
- Games do not have access to `playos-trusted` group (which owns the control IPC socket)
- Verify: `id` inside the game process shows `uid=playos-game`

**Capability dropping:**
- `playos-init` calls `prctl(PR_SET_NO_NEW_PRIVS, 1)` on the game process
- Drop all capabilities from the game's effective and permitted sets before exec
- Preserve only: none required for a normal Raylib game
- `playos-compositor` requires: `CAP_SYS_ADMIN` (DRM master), but games never inherit this

**DRM node access:**
- `/dev/dri/card*` — owned by `drm` group; games are not in `drm` group
- `/dev/dri/renderD*` — needed for GPU rendering via Wayland; games connect through Wayland, not directly
- Verify: `open("/dev/dri/card0", O_RDWR)` returns `EACCES` in the game process

### Input Device Isolation (Reserved Buttons)

**Current gap (pre-Sprint 12):** Reserved buttons (`SYSTEM`, `QUICK_MENU`) are
stripped only by a software bitmask in `libplayos`
(`playos_input_get_controller_state()` does
`state->buttons &= ~(SYSTEM | QUICK_MENU)`). This is a cooperative convention,
not an OS-enforced boundary. A game that opens `/dev/input/event*` directly —
via a Raylib evdev backend or any raw evdev read — can see the reserved
buttons. Today games are spawned with a plain `fork()`+`exec()` from PID 1
(root) with no credential drop, so they *can* open the input devices.

**Required enforcement (this sprint):**
- Games run as `playos-game` (UID ~1000), **not** in the `input` group, so
  `/dev/input/event*` (`root:input`, mode `0660`) is unopenable.
- seccomp `open`/`openat` arg filter and the Landlock allowed-path set both
  **explicitly deny** `/dev/input/event*` (see the seccomp + Landlock sections
  below — the path must be named, not just implied).
- Games receive input only through the Wayland seat; the compositor intercepts
  reserved buttons at the libinput layer and never forwards them to clients
  (see `security-model.md` §8).
- The libplayos software mask becomes defense-in-depth, **not** the sole
  mechanism — the boundary moves to seat/group/permission enforcement.

**Verify:** from a `playos-game` process, `open("/dev/input/event0", O_RDONLY)`
returns `EACCES`, and reserved buttons never appear in the game's input stream.

### seccomp Filter for Games

Apply a seccomp-BPF allowlist to all game processes before exec.

**Allowed syscalls (core set):**
- Memory: `mmap`, `munmap`, `mprotect`, `brk`
- I/O: `read`, `write`, `open`, `openat`, `close`, `stat`, `fstat`, `lseek`, `ioctl` (restricted)
- Sockets: `socket` (AF_UNIX only for Wayland), `connect`, `sendmsg`, `recvmsg`
- Process: `exit`, `exit_group`, `futex`, `clone`, `getpid`, `gettid`
- Signals: `rt_sigaction`, `rt_sigprocmask`, `sigaltstack`
- Time: `clock_gettime`, `clock_nanosleep`
- Misc: `getrandom`, `prctl` (limited)

**Denied syscalls (trigger SIGSYS or EPERM):**
- `mount`, `umount2` — no filesystem mounting
- `init_module`, `finit_module` — no kernel modules
- `ptrace` — no process tracing of other processes
- `reboot` — no direct reboot
- `setuid`, `setgid`, `setcap` — no privilege escalation
- `open`/`openat` with path to `/proc/*/mem`, `/dev/dri/card*`, `/dev/input/event*`, control socket

Generate the seccomp filter at build time. Test with `libseccomp`.

### Landlock Filesystem Restrictions

Apply Landlock rules to game processes:

**Allowed paths:**
```
/data/games/<game-id>/           read-only
/data/saves/<game-id>/           read-write
/data/cache/<game-id>/           read-write
/tmp/ (or /run/game-<id>/)       read-write (scratch space)
/run/playos/ (Wayland socket)    execute only
```

**Denied (by Landlock default — not in allowed set):**
```
/data/games/<other-game>/        no access to other games
/data/saves/<other-game>/        no access to other saves
/data/config/                    no system config access
/run/playos/control.sock         no access to control IPC
/dev/input/event*                no raw input device access
/proc/*/                         no process snooping
```

Landlock requires kernel ≥ 5.13. Verify the ROG Ally kernel version supports it. Fall back to logging-only enforcement if the kernel does not support Landlock (with an alert).

### Control IPC Socket Hardening

- `/run/playos/control.sock` — owned `root:playos-trusted`, mode `0660`
- Only `playos-shell` and `playos-overlay` are in the `playos-trusted` group
- Game processes are never in `playos-trusted`
- Verify: a process running as `playos-game` cannot connect to the control socket

### Production Build — Remove Debug Services

**Must not be present in production builds:**
- BusyBox shell (`/bin/sh`, `/bin/busybox`)
- SSH daemon
- `gdbserver`, `strace`, `ltrace`
- `evtest`, `modetest`, graphics diagnostic tools
- Any open listening TCP/UDP socket

**Build enforcement:**
- Production `defconfig` does not include `BR2_PACKAGE_BUSYBOX` or debug packages
- Post-build script asserts the absence of debug binaries
- CI has a separate "production image lint" step that fails if debug artifacts are present

**Development image retains all debug tools** (behind `BR2_PACKAGE_PLAYOS_DEV_TOOLS`).

### Signed Manifests (Foundations)

Define the manifest signing scheme (full enforcement deferred to post-MVP):
- Games will eventually require a signed `manifest.json`
- For this sprint: define the signing format (Ed25519 detached signature alongside `manifest.json`)
- Implement signature verification in `playos-init` but run in warn-only mode (log if missing/invalid, do not block launch)
- Upgrade to hard-enforcement in a future sprint or post-MVP

### Secure Boot Chain (Documentation + Key Infrastructure)

For this sprint: document the target Secure Boot chain and set up development signing keys.

**Target signed chain:**
1. UEFI Secure Boot — signs `BOOTX64.EFI`
2. `BOOTX64.EFI` — signs or contains the kernel
3. Kernel — verifies the initramfs (dm-verity or IMA)
4. A/B update bundles — signed with PlayOS update key

**Development key setup:**
- Generate a development EFI signing key (self-signed, not trusted by default UEFI)
- Add `sbsign`/`pesign` to the build pipeline
- Production signing uses an HSM-backed key (post-MVP)

---

## Acceptance Criteria

- [ ] Game process runs as `playos-game` user (verified via `playos_system.h` test call or logs)
- [ ] `open("/dev/dri/card0", O_RDWR)` returns `EACCES` in a game process
- [ ] `open("/dev/input/event0", O_RDONLY)` returns `EACCES` in a game process
- [ ] Reserved buttons (`SYSTEM`/`QUICK_MENU`) never appear in a game's input stream (compositor-intercepted, not just libplayos-masked)
- [ ] `connect()` to `/run/playos/control.sock` returns `EACCES` from a `playos-game` process
- [ ] seccomp filter: `mount()` from game process returns `EPERM`
- [ ] Landlock: game cannot `open()` another game's save directory
- [ ] Landlock: game cannot read `/data/config/`
- [ ] Production build contains no `busybox`, `gdbserver`, `strace`, or open TCP sockets
- [ ] Post-build production lint CI step passes
- [ ] Development image retains full debug tools
- [ ] Manifest signature verification runs in warn-only mode (warning in log for unsigned manifests)
- [ ] Development EFI signing key exists and is used in CI builds
- [ ] All existing sprint acceptance criteria still pass (no regression)

---

## Repositories Primarily Involved

| Repo | Work |
|---|---|
| `playos-refdistro` | Game user setup, seccomp filter, Landlock integration, production build lint, debug package gating |
| `playos-platform-api` | `libseccomp` integration, Landlock application before game exec |
| `playos-spec` | Security model documentation, Secure Boot chain spec, manifest signing spec |
| `playos-runtime` | IPC socket permission hardening |

---

## Testing Approach

- Security test suite: small C programs that attempt each forbidden operation; assert they are blocked
- Production build lint: automated check for debug binaries
- Full regression: run all previous acceptance criteria after hardening
- Physical ROG Ally: verify game still launches and functions correctly under all restrictions

---

## Exit Gate

Game processes are privilege-reduced and cannot access trusted IPC, DRM primary nodes, or other games' data. Production builds ship without debug services. Security restrictions do not break any existing functionality.

*Previous: [Sprint 11](Sprint-11.md) | Next: [Sprint 13](Sprint-13.md)*
