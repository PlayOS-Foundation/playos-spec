# PlayOS Security Model

> **Cross-references:** [architecture.md](architecture.md) §13–15, [runtime-ipc.md](runtime-ipc.md) §3, [sprints/Sprint-12.md](sprints/Sprint-12.md)

---

## Table of Contents

1. [Trust Zones](#1-trust-zones)
2. [Component Privilege Levels](#2-component-privilege-levels)
3. [Game Restrictions](#3-game-restrictions)
4. [IPC Access Control](#4-ipc-access-control)
5. [Filesystem Access Control](#5-filesystem-access-control)
6. [seccomp Filter Policy](#6-seccomp-filter-policy)
7. [Landlock Filesystem Restrictions](#7-landlock-filesystem-restrictions)
8. [Input Security](#8-input-security)
9. [System Image Integrity](#9-system-image-integrity)
10. [Secure Boot Chain](#10-secure-boot-chain)
11. [Development vs Production](#11-development-vs-production)
12. [Post-MVP Hardening Roadmap](#12-post-mvp-hardening-roadmap)

---

## 1. Trust Zones

```
┌───────────────────────────────────────────────────────────────┐
│  Zone 1: Kernel                                               │
│  Linux kernel + drivers                                       │
│  Full hardware access                                         │
└───────────────────┬───────────────────────────────────────────┘
                    │
┌───────────────────▼───────────────────────────────────────────┐
│  Zone 2: Trusted System Components                            │
│  playos-init (root)                                           │
│  playos-compositor (display + input caps)                     │
│  playos-shell (playos-trusted group)                          │
│  playos-overlay (playos-trusted group)                        │
│                                                               │
│  ← communicate via /run/playos/ UNIX sockets →               │
└───────────────────┬───────────────────────────────────────────┘
                    │ public libplayos C ABI only
┌───────────────────▼───────────────────────────────────────────┐
│  Zone 3: Untrusted Game Process                               │
│  User: playos-game (unprivileged)                             │
│  No access to Zone 2 sockets                                  │
│  No DRM primary nodes                                         │
│  No raw input devices                                         │
│  Restricted to own save/cache directories                     │
│  seccomp + Landlock enforced                                  │
└───────────────────────────────────────────────────────────────┘
```

---

## 2. Component Privilege Levels

| Component | User | Capabilities | Notes |
|---|---|---|---|
| `playos-init` | `root` | All (required for process supervision, mounts, device setup) | Drop unnecessary caps after init |
| `playos-compositor` | `playos-system` | `CAP_SYS_ADMIN` (DRM master), `CAP_DAC_READ_SEARCH` (device nodes) | Drop all others |
| `playos-shell` | `playos-system` | None | Member of `playos-trusted` group |
| `playos-overlay` | `playos-system` | None | Member of `playos-trusted` group |
| Active game | `playos-game` | None | `PR_SET_NO_NEW_PRIVS = 1` before exec |
| `playos-installer` | `root` | All | Only present in installer image |

---

## 3. Game Restrictions

A normal game **must not** be able to:

| Action | Enforcement |
|---|---|
| Modify the system image | System partition mounted `ro`; game user has no write access |
| Open DRM primary nodes (`/dev/dri/card*`) | `drm` group; game is not in it |
| Access another game's save data | Landlock path restrictions + per-game directory |
| Mount or format filesystems | seccomp blocks `mount`, `umount2` |
| Load kernel modules | seccomp blocks `init_module`, `finit_module` |
| Change kernel parameters | seccomp blocks `sysctl`, game user has no `/proc/sys` write access |
| Invoke unrestricted shutdown/reboot | seccomp blocks `reboot` syscall |
| Connect to control IPC | UNIX group restriction; game is not in `playos-trusted` |
| Synthesize reserved system input | Input routing is compositor-enforced at Wayland/evdev level |
| Create trusted overlays | Trusted roles require `PLAYOS_TRUSTED_*` env + group check |
| Ptrace other processes | seccomp blocks `ptrace` |
| Escalate privileges | `PR_SET_NO_NEW_PRIVS`; seccomp blocks `setuid`, `setcap` |

---

## 4. IPC Access Control

### Control socket (`/run/playos/control.sock`)
- **Owner:** `root:playos-trusted`, mode `0660`
- **Who can connect:** `playos-shell`, `playos-overlay` (both in `playos-trusted`)
- **Who cannot:** `playos-game` — enforced by UNIX group check at `connect(2)` time

### Compositor socket (`/run/playos/compositor.sock`)
- **Owner:** `root:playos-trusted`, mode `0660`
- **Who can connect:** `playos-runtime` internal client only
- **Who cannot:** `playos-game`

### Lifecycle fd (`PLAYOS_LIFECYCLE_FD`)
- **Direction:** Write end held by `playos-init`; read end passed to game
- **Game can:** Read lifecycle events (single-byte values)
- **Game cannot:** Write to the fd; the write end is `close()`d in the game process before exec
- **Not a socket:** Cannot be used to connect to any IPC endpoint

---

## 5. Filesystem Access Control

### System partition (`/`)
- Mounted read-only via `MS_RDONLY` (Sprint 11)
- dm-verity hash tree appended to system image (Sprint 12+ — planned, not yet implemented)
- Any write attempt returns `EROFS`

### Data partition (`/data`)
- Mounted read-write, owned by `root`
- Per-game directories: `chown playos-game:playos-game /data/saves/<id>` and `cache/<id>`
- Other directories (`config/`, `games/`, `logs/`) owned by `playos-system`, not writable by games

### Device nodes
| Device | Owner | Mode | Game access |
|---|---|---|---|
| `/dev/dri/card*` | `root:drm` | `0660` | ❌ Not in `drm` group |
| `/dev/dri/renderD*` | `root:render` | `0660` | ✅ In `render` group (needed for Wayland/EGL) |
| `/dev/input/event*` | `root:input` | `0660` | ❌ Not in `input` group (input via Wayland seat only) |
| `/run/playos/*.sock` | `root:playos-trusted` | `0660` | ❌ Not in `playos-trusted` |

Note: Games access GPU rendering through the Wayland EGL surface, not directly through render nodes.

---

## 6. seccomp Filter Policy

Applied to game processes via `libseccomp` before `execve()`. Default action: `SECCOMP_RET_ERRNO(EPERM)`.

### Allowed syscalls (core game set)

```
# Memory management
mmap, munmap, mprotect, mremap, madvise, brk

# File I/O
read, readv, write, writev, open, openat, close, stat, fstat, lstat,
newfstatat, statx, lseek, dup, dup2, ioctl (restricted — see below),
fcntl, access, faccessat, getdents64, getcwd

# Network (AF_UNIX only for Wayland socket)
socket (AF_UNIX only — enforced by arg filter), connect, bind,
accept, sendmsg, recvmsg, sendto, recvfrom, shutdown, getsockname, getpeername

# Processes and threads
exit, exit_group, clone, clone3, fork, execve (restricted — no setuid),
wait4, waitid, getpid, getppid, gettid, set_tid_address, prctl (restricted)

# Synchronization
futex, futex_waitv, nanosleep, clock_nanosleep

# Signals
rt_sigaction, rt_sigprocmask, rt_sigreturn, sigaltstack, kill (self only)

# Time
clock_gettime, clock_getres, gettimeofday, time

# Misc
getrandom, getuid, getgid, geteuid, getegid, uname, sysinfo,
pread64, pwrite64, eventfd2, epoll_create1, epoll_ctl, epoll_wait,
pipe2, timerfd_create, timerfd_settime, timerfd_gettime,
inotify_init1, inotify_add_watch, inotify_rm_watch,
mlock, munlock, memfd_create
```

### Blocked syscalls (fatal SIGSYS or EPERM)

```
mount, umount2, umount           # no filesystem mounting
init_module, finit_module, delete_module  # no kernel modules
reboot                           # no direct reboot
ptrace                           # no process tracing
setuid, setgid, setresuid, setresgid, setfsuid, setfsgid  # no privilege escalation
capset, prctl(PR_SET_SECCOMP)    # no capability changes
sysctl, nfsservctl               # no kernel parameter changes
kexec_load, kexec_file_load      # no kernel replacement
iopl, ioperm                     # no direct I/O port access
perf_event_open                  # no performance counters (in retail builds)
```

### ioctl restrictions
`ioctl` is allowed only with these device categories:
- Wayland socket (AF_UNIX)
- `/dev/dri/renderD*` (DRI render node — needed for EGL)
- `/dev/dri/card*` — **blocked** (prevents DRM master)

---

## 7. Landlock Filesystem Restrictions

Requires Linux ≥ 5.13 (ROG Ally ships with kernels that support this). Falls back to logging-only if unavailable.

### Allowed paths for game processes

| Path | Access |
|---|---|
| `/data/games/<game-id>/` | Read-only |
| `/data/saves/<game-id>/` | Read + write + create + remove |
| `/data/cache/<game-id>/` | Read + write + create + remove |
| `/run/playos/playos-0` (Wayland socket) | Connect (execute) |
| `/dev/dri/renderD*` | Read (for EGL) |
| `/dev/urandom`, `/dev/random` | Read |
| `/proc/self/` | Read-only |
| `/tmp/game-<id>/` | Read + write + create + remove |

### Denied (implicitly — not in allowed set)

- `/data/games/<other-game-id>/` — other games
- `/data/saves/<other-game-id>/` — other games' saves
- `/data/config/` — system configuration
- `/data/log/` — system logs (games write via `playos_log()`, not direct fs access)
- `/run/playos/control.sock` — control IPC
- `/run/playos/compositor.sock` — compositor control
- `/sys/`, `/proc/<other-pid>/` — system and process snooping
- `/dev/dri/card*` — DRM primary nodes

---

## 8. Input Security

> **Current gap (pre-Sprint 12):** The target model below is **not yet
> implemented**. Today reserved buttons are stripped only by a software
> bitmask in `libplayos` (`playos_input.c`), and games are spawned via a plain
> `fork()`+`exec()` from PID 1 (root) with no credential drop, so a game can
> open `/dev/input/event*` directly and read the reserved buttons — bypassing
> the mask. Sprint 12 closes this gap (see `Sprint-12.md` §Input Device
> Isolation).

**Reserved system actions** (`PLAYOS_BUTTON_SYSTEM`, `PLAYOS_BUTTON_QUICK_MENU`) are intercepted at the Wayland compositor's libinput layer before any event reaches a client. They are never present in the game's input stream.

**Input routing hierarchy:**
```
libinput event
    │
    ▼ playos-compositor intercepts
    ├── reserved action → PlayOS only (never to client)
    ├── overlay visible → overlay Wayland client
    ├── game foreground → game Wayland client (filtered: no reserved keys)
    └── otherwise      → shell Wayland client
```

Games receive input exclusively through the **Wayland seat** (not raw evdev). They cannot open `/dev/input/event*` (not in the `input` group; Landlock also blocks the path).

---

## 9. System Image Integrity

### Sprint 11 (initial): Read-only mount
```
mount(device, "/", "ext4", MS_RDONLY, NULL);
```
Any write to the system partition returns `EROFS`.

### Post-Sprint 12 (production): dm-verity
```bash
# At build time (in playos-refdistro release pipeline):
veritysetup format system.img system.img.verity > system.verity.superblock
```

At boot, `playos-init`:
1. Sets up a dm-verity device over the system partition
2. Mounts the dm-verity device read-only
3. If hash verification fails for any block, the kernel returns I/O errors (enforced by dm-verity)
4. `playos-init` monitors for dm-verity errors; repeated errors trigger A/B rollback

---

## 10. Secure Boot Chain

### Target signing chain (post-MVP, Sprint 12 foundations)

```
UEFI Secure Boot (platform key)
    └── signs BOOTX64.EFI

BOOTX64.EFI (Linux EFI stub)
    └── kernel + embedded initramfs (verified by EFI stub signature)

Kernel (IMA or dm-verity)
    └── system partition hash tree (dm-verity root hash embedded in initramfs)

A/B update bundles
    └── signed with PlayOS update key (RAUC bundle signature)
```

### Development key setup (Sprint 12)
- Self-signed certificate used for development and CI builds
- Production: HSM-backed key, never leaves the signing server
- `sbsign` used in the release pipeline

### Recovery
If Secure Boot verification fails:
1. UEFI firmware refuses to boot the artifact
2. User must boot into UEFI Secure Boot key management to enroll the PlayOS development key (dev builds)
3. Production: chain-of-trust failure surfaces as boot failure → A/B rollback → recovery mode

---

## 11. Development vs Production

| Feature | Development image | Production image |
|---|---|---|
| BusyBox shell | ✅ Present | ❌ Absent |
| SSH daemon | ❌ (planned post-network sprint) | ❌ Absent |
| `gdbserver`, `strace` | ✅ Present | ❌ Absent |
| Serial console | ✅ Enabled | ✅ Enabled (needed for recovery) |
| dm-verity | Optional | ✅ Required |
| Secure Boot | Optional (disabled ok) | ✅ Required |
| Debug assertions | ✅ Enabled | ❌ Disabled |
| Open TCP sockets | Allowed (SSH) | ❌ None |
| seccomp | ✅ Enforced | ✅ Enforced |
| Landlock | ✅ Enforced | ✅ Enforced |

The post-build production lint CI step asserts:
- No `/bin/sh` or `/bin/busybox` in the image
- No open listening TCP sockets
- No `gdbserver`, `strace`, or debug tools
- System partition is read-only
- All EFI artifacts are signed

---

## 12. Post-MVP Hardening Roadmap

In priority order after v0.1.0:

1. **dm-verity** for system partition integrity (Sprint 12 gap)
2. **Signed game manifests** (Ed25519 — warn-only in Sprint 12, enforced post-MVP)
3. **User namespaces** for additional game isolation if needed
4. **Hardware-backed keys** for update signing
5. **IMA/EVM** for individual file integrity in the initramfs
6. **Audit logging** for privileged IPC commands
7. **Network namespace** for games (when networking is introduced)
8. **Mandatory access control** (SELinux or AppArmor) if seccomp + Landlock proves insufficient
