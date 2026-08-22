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
| `playos-init` | `root` | All (required for process supervision, mounts, device setup) | Drop unnecessary caps after init (future) |
| `playos-compositor` | `root` | All | Sprint 12 scopes privilege reduction to games; dropping system-component privileges is deferred |
| `playos-shell` | `root` | All | Trusted; root accepted by control-socket check |
| `playos-overlay` | `root` | All | Trusted; root accepted by control-socket check |
| Active game | `playos-game` (uid 1001, gid 1001) | None | `PR_SET_NO_NEW_PRIVS = 1`; supplementary group `audio` only; seccomp + Landlock before exec |
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

Applied to game processes as a hand-built classic-BPF filter (no runtime
`libseccomp` dependency — `playos-init` is a static PID 1). The filter
verifies `AUDIT_ARCH_X86_64`, kills any other ABI, and returns `EPERM`
for the privileged/credential syscall **deny-list** below. A full syscall
*allowlist* is deferred (games are dynamically linked; see Sprint-12.md
Decisions). Path-based `open`/`openat` restrictions are enforced by
Landlock (§7), not by pointer-dereferencing BPF.

### Allowed syscalls

Everything not listed below is allowed. This is a deliberate MVP
tradeoff: Landlock default-deny is the path boundary, the seccomp
deny-list blocks the privileged syscalls Landlock cannot reach.

### Blocked syscalls (return `EPERM`)

```
mount, umount2, pivot_root, chroot, acct, swapon, swapoff, quotactl,
_sysctl, uselib, init_module, finit_module, delete_module
setuid, setgid, setreuid, setregid, setresuid, setresgid, setfsuid,
setfsgid, capset
ptrace, process_vm_readv, process_vm_writev
reboot, kexec_load, kexec_file_load, settimeofday, clock_settime,
adjtimex, sethostname, setdomainname, iopl, ioperm, personality,
vhangup, mknod, unshare, setns
seccomp, bpf, perf_event_open, userfaultfd, add_key, request_key,
keyctl, name_to_handle_at, open_by_handle_at
prctl(PR_SET_SECCOMP)   # games cannot stack/replace filters; other
                        # prctl calls are allowed
```

### ioctl restrictions

Device-node access is enforced by Landlock + group membership, not by
`ioctl` filtering: `/dev/dri/*` and `/dev/input/*` are outside the
Landlock allowlist and the game is not in `drm`/`input`.

---

## 7. Landlock Filesystem Restrictions

Requires Linux ≥ 5.13 (ROG Ally ships with kernels that support this). Falls back to logging-only if unavailable. Implemented in `playos-init/src/security/landlock.c`; rule construction is data-driven by launch identity.

### Allowed paths for game processes

| Path | Access |
|---|---|
| `/data/games/<game-id>/` | Read + execute (read-only game content, dynamic traversal) |
| `/data/saves/<game-id>/` | Read + write + create + remove + truncate |
| `/data/cache/<game-id>/` | Read + write + create + remove + truncate |
| `/tmp` | Read + write + create + remove (shared scratch) |
| `/run/playos/` | Read + execute (Wayland socket path) |
| `/lib`, `/usr/lib` | Read + execute (musl dynamic loader + shared libraries) |
| `/dev/snd` | Read + write (ALSA PCM/control nodes) |
| `/dev/shm` | Read + write + create + remove (wl_shm fallback) |
| `/etc/asound.conf` | Read (single-file rule, optional) |

### Denied (implicitly — not in allowed set)

- `/data/games/<other-game-id>/` — other games
- `/data/saves/<other-game-id>/` — other games' saves
- `/data/config/` — system configuration
- `/data/log/` — system logs (games write via `playos_log()`, not direct fs access)
- `/run/playos/control.sock` — control IPC
- `/run/playos/compositor.sock` — compositor control
- `/sys/`, `/proc/` — system and process snooping
- `/dev/dri/*` — all DRM nodes (primary **and** render nodes; games reach the GPU through the Wayland seat, so no direct render-node access is granted)
- `/dev/input/event*` — raw input devices

---

## 8. Input Security

**Implemented (Sprint 12).** Reserved system actions (`PLAYOS_BUTTON_SYSTEM`:
`BTN_MODE`/`KEY_PROG1`/`BTN_TRIGGER_HAPPY1`; `PLAYOS_BUTTON_QUICK_MENU`:
`KEY_PROG2`/`BTN_TRIGGER_HAPPY2`) are consumed by `playos-compositor` at the
seat layer (`src/system_button.c`) before any event reaches a client. Games
are additionally denied raw evdev: they run as `playos-game` (not in
`input`), and Landlock default-deny blocks `/dev/input/event*`. The
`libplayos` bitmask remains defense-in-depth only.

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
- Dev keys live in `playos-refdistro/keys/dev/`:
  - `efi-signing-key.pem` + `efi-signing-cert.pem` — self-signed EFI
    signing key used by `scripts/sign-efi.sh` (`sbsign`) for development
    and CI builds
  - `manifest-key.pub` / `manifest-key.sec` — Ed25519 game-manifest
    signing key used by `scripts/sign-manifest.sh`; the public key is
    embedded in `playos-init` (`src/security/game_key.h`) for warn-only
    manifest verification
- Production: HSM-backed keys, never leave the signing server (post-MVP)
- `sbsign` is the release-pipeline signing tool; `pesign` is an
  acceptable alternative for RPM-oriented flows

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
