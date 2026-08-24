# Sprint 12 — Security Hardening

**Goal:** Establish a hardened boundary between the public `playos-platform-api`, trusted `playos-runtime` control paths, and untrusted game processes. Games operate with minimal privileges. Remote debug services are removed from production builds. Secure Boot signing chain is defined.

**Primary Outcome:** A game process cannot access trusted IPC endpoints, cannot open DRM primary nodes, cannot write outside its own save/cache directories, and cannot synthesize reserved system input. Production builds ship without a shell or debug services.

**Prerequisites:** Sprint 11 complete — immutable images and A/B updates working.

**Status:** 🟢 Complete — all T1–T9 implemented and validated. The on-device (ROG Ally) acceptance run surfaced one real gap: Landlock was not active because the Ally kernel had `CONFIG_SECURITY` disabled (and `CONFIG_SECURITY_LANDLOCK` depends on `SECURITY`), so `config_denied` failed (a game could read `/data/config`). Fixed by enabling `CONFIG_SECURITY` + `CONFIG_SECURITY_LANDLOCK` in `br2-external/board/ally/linux.config`; the rebuilt kernel `.config` now shows `CONFIG_SECURITY=y`, `CONFIG_SECURITY_LANDLOCK=y`, and `CONFIG_LSM="landlock,lockdown,yama,loadpin,safesetid,ipe,bpf"`. After re-flashing the rebuilt image, the on-device security self-test passed **11/11** on the ROG Ally (2026-08-23), confirming Landlock default-deny, the seccomp deny-list, credential drop, DRM/input/control-socket boundaries, and cross-game data isolation on real hardware. Production-lint CI is satisfied by `playos-refdistro/.github/workflows/production-build.yml` (builds the production image, asserts no debug artifacts, QEMU boot-checks `--production`).

---

## Why This Sprint Exists

Before Sprint 12, the security boundary is mostly a convention. Games are spawned by `playos-init` (PID 1, root) with a plain `fork()` + `exec()` and no credential drop, so a game can open `/dev/input/event*` directly and read reserved buttons, and the reserved-button protection is only a cooperative bitmask in `libplayos`. Production builds still carry a shell and debug tools. This sprint moves the boundary from convention to OS enforcement: an unprivileged game identity, capability and syscall restrictions, filesystem isolation, hardened control IPC, and debug-free production images.

---

## Start Condition Checklist

- Sprint 11 complete: the system image is read-only and A/B updates work.
- `playos-init` currently spawns games as root with no credential drop (verified gap).
- Reserved buttons are currently stripped only by a software mask in `libplayos` (`playos_input_get_controller_state()`), not OS-enforced.
- `/run/playos/control.sock` exists and is the trusted control path.
- Production `defconfig` still includes debug tools (BusyBox, `gdbserver`, `strace`, etc.).
- ROG Ally kernel version is checked for Landlock support (kernel ≥ 5.13).

---

## Decisions Locked for This Sprint

- **Game identity:** games run as `playos-game` (UID 1001) with three supplementary groups — `audio` (ALSA `/dev/snd` access), `render` (unprivileged DRM render node `/dev/dri/renderD*` for client-side EGL/GLES2), and `input` (`/dev/input/event*` for the built-in controller, which `libplayos` reads directly via its evdev backend). They are intentionally **not** in `video` or `drm` (the `drm` group gates the primary node `/dev/dri/card*`). No per-profile Linux uid is introduced; console-style local profiles are a future data-path layer over this single identity.
- **Sandbox parameterization:** the sandbox path policy is parameterized by launch identity (game id now; a profile id later), so a future profile can be inserted without reworking enforcement.
- **Capability set:** games start with no capabilities; `prctl(PR_SET_NO_NEW_PRIVS, 1)` is set before exec.
- **DRM access:** games render client-side buffers via the DRM *render node* `/dev/dri/renderD*` (granted through `render` group membership), and the compositor scans them out through the Wayland seat. Direct `/dev/dri/card*` (primary node) access stays denied — a game can never modeset or take over the display.
- **Input boundary:** reserved buttons are kept out of the game-facing input stream by two layers — the `libplayos` snapshot mask (`playos_input.c` strips `SYSTEM`/`QUICK_MENU`/`POWER` from `state->buttons`) and the compositor's seat-level listener (`system_button.c`) which consumes the same keycodes before any client can receive them. The game's normal input path is raw evdev (`/dev/input` is granted read-only so the built-in controller works); the reserved-button boundary is enforced by that mask + seat intercept, not by denying `/dev/input`.
- **Sandbox mechanism:** a Landlock filesystem allowlist (default-deny) plus a seccomp-BPF deny-list of privileged/credential syscalls, both applied before game exec. A full seccomp *allowlist* is explicitly deferred: games are dynamically linked (musl + shared libraylib/libplayos), and a hand-maintained allowlist is too regression-prone for the MVP; Landlock's default-deny provides the path-based boundary and the seccomp deny-list blocks the privileged syscalls Landlock cannot reach.
- **Control socket:** `/run/playos/control.sock` is `root:playos-trusted`, mode `0660`; only `playos-shell` and `playos-overlay` are in `playos-trusted`.
- **Manifest signing:** Ed25519 detached signature alongside `manifest.json`; verification is warn-only this sprint.
- **Secure Boot:** documentation plus development signing keys only; production HSM-backed signing is post-MVP.

---

## Scope

### In Scope

- Game process privilege reduction and credential drop in `playos-init`.
- seccomp syscall allowlist and Landlock filesystem allowlist for games.
- Denial of direct DRM primary-node access, and reserved-button input isolation.
- Control IPC socket ownership and permission hardening.
- Removal of debug tools/services from the production image plus CI lint.
- Signed-manifest foundations (warn-only verification).
- Secure Boot chain documentation and development signing keys.

### Explicitly Out of Scope

- Hard enforcement of signed game manifests (post-MVP).
- Production HSM-backed Secure Boot signing (post-MVP).
- Full dm-verity/IMA verified-boot implementation (documented as a target only).
- Network sandboxing (no network stack in the MVP).
- Store-level signing and distribution.
- Multi-user local profiles (deferred to [Sprint 21](Sprint-21.md)); this sprint only establishes the single unprivileged identity and data-driven sandbox they will build on.

---

## Required Repository Changes

| Repo | Required work |
|---|---|
| `playos-init` | Spawn games as `playos-game` with `PR_SET_NO_NEW_PRIVS` and no capabilities; apply seccomp allowlist and Landlock ruleset; verify manifest signatures in warn-only mode |
| `playos-compositor` | Intercept reserved buttons at the libinput/seat layer and never forward them to clients |
| `playos-runtime` | Enforce control socket ownership and permissions (`root:playos-trusted`, `0660`) |
| `playos-refdistro` | Create `playos-game` user, integrate seccomp/Landlock into the game spawn path, remove debug tools from the production `defconfig`, add post-build lint, wire `sbsign`/`pesign` and development EFI keys |
| `playos-spec` | Document the security model, Secure Boot chain, and manifest signing scheme |

---

## Expected Files and Directories

### `playos-init`

```text
src/security/
    sandbox.c                   # PR_SET_NO_NEW_PRIVS, capability drop, setuid to playos-game
    seccomp_filter.c            # build-time generated seccomp-BPF allowlist
    landlock.c                  # Landlock allowed/denied path ruleset
    manifest_verify.c           # Ed25519 detached-signature check (warn-only)
```

### `playos-compositor`

```text
src/system_button.c             # reserved-button interception at the libinput/seat layer
```

### `playos-refdistro`

```text
br2-external/board/ally/users-table.txt          # playos-game UID 1001, groups audio,render,input
br2-external/board/common/rootfs-overlay/etc/udev/rules.d/99-playos-dri.rules  # renderD* -> root:render 0660
br2-external/board/common/rootfs-overlay/etc/udev/rules.d/99-playos-input.rules  # ASUS EC (0b05:1abe) -> root:root 0600
br2-external/board/ally/post-build.sh            # production lint: assert no debug binaries
br2-external/configs/playos_ally_production_defconfig
scripts/sign-efi.sh                              # sbsign/pesign with the development key
keys/dev/                                        # development EFI signing key (not production HSM)
```

### `playos-spec`

```text
src/security-model.md           # updated: §8 reserved-button boundary, sandbox, Secure Boot chain
```

---

## Agent Task Breakdown

### Task Status Grid

| Task ID | Task | Primary repo | Status | Notes / evidence |
|---|---|---|---|---|
| S12-T1 | Drop game privileges at spawn: UID 1001, `PR_SET_NO_NEW_PRIVS`, capability drop | `playos-init` | done | `src/security/sandbox.c`; hard-fails launch if drop fails |
| S12-T2 | Apply Landlock allowed/denied path policy to game processes | `playos-init` | done | `src/security/landlock.c`; host test: allowlist honored, default-deny enforced; unsupported-kernel fallback logs and continues |
| S12-T3 | Apply seccomp syscall deny-list (see Decisions) | `playos-init` | done | `src/security/seccomp_filter.c`; host test skips in seccomp-blocked sandbox, exercised on-device |
| S12-T4 | Deny DRM primary-node access from games | `playos-init` | done | Primary node `/dev/dri/card*` denied (games not in `drm`); render node `/dev/dri/renderD*` granted via `render` group + `99-playos-dri.rules` + a `/dev/dri` read-write Landlock rule for client-side EGL |
| S12-T5 | Enforce reserved-button input isolation end-to-end | `playos-compositor`, `playos-platform-api`, `playos-init`, `playos-refdistro` | done | Compositor seat intercept extended to BTN_MODE/KEY_PROG1/BTN_TRIGGER_HAPPY1/KEY_PROG2/BTN_TRIGGER_HAPPY2; `libplayos` snapshot mask strips reserved buttons; `/dev/input` granted read-only for the gamepad, and `99-playos-input.rules` moves the ASUS EC (`0b05:1abe`) to `root:root 0600` so games cannot read Command Center/M1-M2 directly |
| S12-T6 | Harden `control.sock` trusted-client auth and permission checks | `playos-runtime`, `playos-init` | done | Socket already `0660 root:1000` + `SO_PEERCRED` gid/uid check; policy test added in `playos-runtime` |
| S12-T7 | Strip debug tools/services from the production image | `playos-refdistro` | done | Production defconfig drops BusyBox/sh/init; `post-build.sh` lint passes (`production lint: OK — no debug artifacts`); target verified free of busybox/sh/gdbserver/strace/evtest/dropbear/sshd; QEMU boots to PID 1 without a shell |
| S12-T8 | Verify signed game manifests (warn-only in MVP) | `playos-init`, `playos-runtime` | done | Self-contained Ed25519 in `playos-init`; RFC 8032 + Python cross-verify; `scripts/sign-manifest.sh`; warn-only in spawn path |
| S12-T9 | Document Secure Boot chain; create and rotate dev signing keys in image build | `playos-refdistro`, `playos-spec` | done | `keys/dev/*` + `scripts/sign-efi.sh` created; `security-model.md` §10 Secure Boot Chain documented; production HSM signing deferred |

Update the **Status** column as work progresses: `not started` → `in progress` → `blocked` or `done`.

### S12-T1 — Drop game privileges at spawn

`playos-init` spawns games as the `playos-game` user (UID 1001) with supplementary groups `render` (108), `audio` (29), and `input` (102), calls `prctl(PR_SET_NO_NEW_PRIVS, 1)` on the game process, and drops all capabilities from the effective and permitted sets before `exec`. The `playos-compositor` keeps `CAP_SYS_ADMIN` (DRM master), but games never inherit it.

**Done when:** from inside a running game, `id` shows `uid=playos-game`, `/proc/self/status` shows `NoNewPrivs: 1`, and `CapEff: 0000000000000000`.

### S12-T2 — Apply Landlock filesystem restrictions

Build a Landlock ruleset granting exactly the paths a game needs and default-deny everything else. The MVP allowlist is:

- `/data/games/<game-id>/` — read-only (`READ_FILE | EXECUTE`)
- `/data/saves/<game-id>/` and `/data/cache/<game-id>/` — read-write, including file/dir creation and removal (`READ_FILE | WRITE_FILE | TRUNCATE | MAKE_REG | MAKE_DIR | REMOVE_FILE | REMOVE_DIR`)
- `/tmp` — read-write scratch (`READ_FILE | WRITE_FILE | TRUNCATE | MAKE_REG | MAKE_DIR | REMOVE_FILE | REMOVE_DIR`)
- `/run/playos/` — `READ_FILE | EXECUTE` for the Wayland socket path
- `/lib` and `/usr/lib` — `READ_FILE | EXECUTE` for the musl dynamic loader and shared libraries (games are dynamically linked against `libraylib`/`libplayos`; without this the exec of any sample game fails with `ENOENT` on the interpreter)
- `/dev/snd` — `READ_FILE | WRITE_FILE` for ALSA PCM/control nodes (audio must keep working)
- `/dev/input` — `READ_FILE` for the built-in controller (libplayos evdev backend); reserved buttons are stripped by the `libplayos` mask + compositor seat intercept
- `/dev/dri` — `READ_FILE | WRITE_FILE` for the DRM render node (`/dev/dri/renderD*`), which client-side EGL/GLES2 opens `O_RDWR`; the primary node `/dev/dri/card*` stays denied by the `drm` group (Unix DAC)
- `/dev/shm` — `READ_FILE | WRITE_FILE | TRUNCATE | MAKE_REG | REMOVE_FILE` for Wayland `wl_shm` fallback
- `/etc/asound.conf` — `READ_FILE` (single-file rule, best-effort if present)

Deny by default (no rule, so denied): other games' data, `/data/config/`, `/run/playos/control.sock`, `/dev/dri/card*`, `/proc/*/`. Build the ruleset from launch-time variables (game id now; a `profile-id` prefix later) so Sprint 21 can add profile scoping by changing one path-construction function, not the enforcement logic. If the kernel is older than 5.13, fall back to logging-only enforcement with an alert.

**Done when:** a game process cannot `open()` another game's save directory or read `/data/config/`; the unsupported-kernel fallback logs an alert without blocking launch; existing sample games still boot (dynamic loader + ALSA paths granted).

### S12-T3 — Apply seccomp syscall restrictions

Apply a seccomp-BPF filter (constructed at spawn time, before `exec`) that denies the privileged and credential syscalls a game must never use: `mount`, `umount2`, `init_module`, `finit_module`, `delete_module`, `ptrace`, `reboot`, `kexec_load`, `kexec_file_load`, `swapon`, `swapoff`, `setuid`, `setgid`, `setreuid`, `setregid`, `setresuid`, `setresgid`, `setfsuid`, `setfsgid`, `capset`, `chroot`, `pivot_root`, `acct`, `settimeofday`, `clock_settime`, `adjtimex`, `sethostname`, `setdomainname`, `personality`, `iopl`, `ioperm`, `bpf`, `perf_event_open`, `process_vm_readv`, `process_vm_writev`, `userfaultfd`, `add_key`, `request_key`, `keyctl`, `open_by_handle_at`, `name_to_handle_at`, and `seccomp` itself. `prctl` is allowed except `prctl(PR_SET_SECCOMP)`, which is denied (arg-checked). The filter verifies `AUDIT_ARCH_X86_64` and kills on any other ABI. Path-based `open`/`openat` restrictions (`/proc/*/mem`, `/dev/dri/card*`, control socket) are enforced by Landlock (T2), not by pointer-dereferencing BPF.

**Done when:** `mount()` from a game process returns `EPERM`, `prctl(PR_SET_SECCOMP)` returns `EPERM`, and `open()` of `/proc/*/mem`, `/dev/dri/card*`, or the control socket is denied (by Landlock).

### S12-T4 — Restrict DRM node access

The **primary node** `/dev/dri/card*` is owned by the `drm` group; games are not in `drm`, so they cannot modeset or take over the display. The **render node** `/dev/dri/renderD*` is a separate, unprivileged GPU path that games legitimately need for client-side EGL/GLES2 (raylib's PLAYOS backend renders via EGL + DRI, then the compositor scans the buffers out through the Wayland seat). A `99-playos-dri.rules` udev rule sets render nodes to `root:render 0660`, `playos-game` is a member of `render`, and the Landlock allowlist grants `/dev/dri` read-write (the primary node stays denied by the `drm` group, not by Landlock). Verify the primary node is unopenable from the game identity while the render node is openable.

**Done when:** `open("/dev/dri/card0", O_RDWR)` from a game process returns `EACCES`, and `open("/dev/dri/renderD128", O_RDWR)` succeeds so the game's EGL context initializes.

### S12-T5 — Enforce reserved-button input isolation

Reserved buttons (`SYSTEM`, `QUICK_MENU`, `POWER`) are stripped from the game-facing input stream by two layers plus a udev boundary: `playos-platform-api`'s snapshot mask (`playos_input.c`) clears `PLAYOS_RESERVED_BUTTONS` from `state->buttons`, and `playos-compositor`'s seat-level listener (`system_button.c`) consumes the same keycodes before any client can receive them. The gamepad node (`045e:028e`) stays in `input` and `/dev/input` is granted read-only so the built-in controller (raw evdev via `libplayos`) works. `99-playos-input.rules` additionally moves the ASUS embedded-controller node (`0b05:1abe`, `event6/7/8`) to `root:root 0600`, so a game in `input` cannot open it and read Command Center (`KEY_PROG1`/`KEY_PROG2`) or M1/M2 directly; the compositor and shell read those as root.

**Done when:** `SYSTEM`/`QUICK_MENU` never appear in a game's input stream, pressing them while a game is focused still drives shell/overlay behavior, and `open("/dev/input/event8")` from a game process returns `EACCES`.

### S12-T6 — Harden control socket

`/run/playos/control.sock` is owned by `root:playos-trusted` with mode `0660`. Only `playos-shell` and `playos-overlay` are in the `playos-trusted` group; game processes are never in that group.

**Done when:** `connect()` to `/run/playos/control.sock` from a `playos-game` process returns `EACCES`.

### S12-T7 — Remove debug tools from production build

The production `defconfig` excludes BusyBox (`/bin/sh`, `/bin/busybox`), the SSH daemon, `gdbserver`/`strace`/`ltrace`, `evtest`/`modetest` and graphics diagnostic tools, and ships with no open listening TCP/UDP sockets. A post-build script asserts their absence, and CI has a production-image lint step that fails if debug artifacts are present. The development image retains all debug tools behind `BR2_PACKAGE_PLAYOS_DEV_TOOLS`. BusyBox is NOT init (PID 1 is `playos-init`, installed at `/init`), so excluding BusyBox removes `/bin/sh` and its applets only; `ip` (BusyBox) is replaced by `iproute2` and `udhcpc` (BusyBox) by `dhcpcd`.

**Open decision — future Developer Mode SSH:** if Dropbear is ever re-enabled on a BusyBox-free production/developer image, an interactive SSH login still needs a shell to `exec` (Dropbear reads it from `/etc/passwd`). Choose between shipping a minimal standalone shell (`mksh`/`dash`) or a `command=`-style forced command — see `post-mvp.md` §SSH Developer Mode.

**Done when:** the production image contains no `busybox`, `gdbserver`, or `strace` and no open sockets; the CI lint step fails if any of these appear.

### S12-T8 — Verify signed game manifests (warn-only)

Define the manifest signing format: an Ed25519 detached signature in raw 64-byte form at `<manifest>.sig` (e.g. `/data/games/<id>/manifest.json.sig`). Implement Ed25519 signature verification in `playos-init` (self-contained, no libsodium/libseccomp dependency — static PID 1). The verifier uses the PlayOS development public key embedded at build time (`keys/dev/manifest-key.pub`, 32 raw bytes); a matching signing script lives in `playos-refdistro/scripts/sign-manifest.sh`. Verification runs in warn-only mode — log a warning if the signature is missing or invalid, and do not block launch. Hard enforcement is deferred to a later sprint or post-MVP.

**Done when:** an unsigned manifest produces a warning in the log and the game still launches; a validly signed manifest verifies without warnings; a corrupted signature produces a warning but still launches.

### S12-T9 — Document Secure Boot chain and development keys

Document the target signed chain: UEFI Secure Boot signs `BOOTX64.EFI`; `BOOTX64.EFI` signs or contains the kernel; the kernel verifies the initramfs via dm-verity or IMA; A/B update bundles are signed with the PlayOS update key. Generate a self-signed development EFI signing key and add `sbsign`/`pesign` to the build pipeline. Production signing uses an HSM-backed key (post-MVP).

**Done when:** the development EFI key exists under `keys/dev/` and is used by CI builds, `security-model.md` documents the chain, and production HSM signing is explicitly deferred.

---

## Implementation Guidance

**Drop privileges before exec, not after.** The game must never execute with root credentials; perform the credential drop and `PR_SET_NO_NEW_PRIVS` in the child process before `execve`.

**Name the denied paths explicitly.** The Landlock ruleset denies `/dev/dri/card*`, `/proc/*/mem`, and the control socket (default-deny); `/dev/input` is intentionally granted read-only for the controller, and reserved-button isolation is enforced by the snapshot mask + seat intercept, not by path denial. The seccomp filter is a syscall deny-list only — it does not filter by path.

**Treat the `libplayos` mask as one of two enforcement points.** Reserved-button isolation is the snapshot mask plus the compositor seat intercept; both strip the same reserved keycodes. Neither is "defense-in-depth only" — together they are the boundary, because the game legitimately needs `/dev/input` for the controller.

**Landlock fallback must be loud, not silent.** If the kernel is too old, log an alert and continue, but record it as an unresolved hardening gap rather than failing the whole launch path.

**Keep Landlock path construction data-driven.** Build paths in one function taking the launch identity (game id now, profile id later) so the ruleset does not encode raw path strings and Sprint 21 can add profile scoping without touching enforcement.

**Production lint fails the build.** Debug-artifact absence is enforced in CI, not checked manually.

**Keep Secure Boot scoped this sprint.** Ship documentation and development keys only; do not attempt production key management.

---

## Verification and Evidence

| Evidence | How it is produced |
|---|---|
| Game runs as `playos-game` with no capabilities | `id` and `/proc/self/status` captured inside a test game |
| Primary DRM node denied | Security test binary attempts `open("/dev/dri/card0", O_RDWR)` |
| Raw input device granted (controller works) | Security test binary opens `/dev/input/event0` read-only (input group + Landlock rule) |
| Reserved buttons absent | Input-stream dump while `SYSTEM`/`QUICK_MENU` are pressed |
| Control socket denied | Security test binary attempts `connect()` to `control.sock` |
| seccomp active | Security test binary attempts `mount()` and observes `EPERM` |
| Landlock active | Security test binary reads another game's save directory and is denied |
| Production image is debug-free | CI production lint log and post-build script output |
| Manifest warn-only behavior | Launch log for an unsigned manifest |
| Secure Boot chain documented | `security-model.md` plus `keys/dev/` in CI artifacts |

---

## Acceptance Criteria

- [x] Game process runs as `playos-game` user (host: `playos_security_drop_privileges()` unit path; on-device self-test `identity` PASS)
- [x] `open("/dev/dri/card0", O_RDWR)` returns `EACCES` in a game process (games not in `drm` group — Unix DAC; host Landlock test covers the `/dev/dri` render-node grant; on-device self-test `dri_card0_denied`/`dri_render_allowed` PASS)
- [x] `open("/dev/input/event0", O_RDONLY)` succeeds in a game process so the controller works (input group + Landlock `/dev/input` read rule; on-device self-test `input_allowed` PASS)
- [x] Reserved buttons (`SYSTEM`/`QUICK_MENU`) never appear in a game's input stream (compositor seat intercept + `libplayos` snapshot mask; on-device manual smoke PASS — reserved presses drive shell/overlay, never game input)
- [x] `connect()` to `/run/playos/control.sock` returns `EACCES` from a `playos-game` process (socket `0660 root:1000`; policy test added)
- [x] seccomp filter: `mount()` from game process returns `EPERM` (filter built; host env blocks filter installation — on-device self-test `mount_denied` PASS)
- [x] Landlock: game cannot `open()` another game's save directory (host test)
- [x] Landlock: game cannot read `/data/config/` (default-deny; host test)
- [x] Production build contains no `busybox`, `gdbserver`, `strace`, or open TCP sockets (lint `OK — no debug artifacts` + manual absence of busybox/sh/gdbserver/strace/evtest/dropbear/sshd; no listening network services by construction since dropbear/sshd are removed)
- [x] Post-build production lint CI step passes (`playos-refdistro/.github/workflows/production-build.yml` runs `make ally-production-build`, asserts no debug artifacts, and QEMU boot-checks `--production`)
- [x] Development image retains full debug tools (`playos_ally_defconfig` unchanged debug set)
- [x] Manifest signature verification runs in warn-only mode (warning in log for unsigned manifests — spawn path logs, launch never blocked)
- [x] Development EFI signing key exists and is used in CI builds (`keys/dev/*`, `scripts/sign-efi.sh`; CI wiring is part of T7 image build)
- [x] All existing sprint acceptance criteria still pass (no regression — QEMU boot smoke test passes to `playos-init starting as PID 1` with no busybox/sh; host + QEMU regression passes; **on-device 11/11 self-test PASSED after re-flashing the rebuilt image (2026-08-23)** — the Landlock gap was found on-device, the kernel-config fix was rebuilt, re-flashed, and re-verified)

---

## Handoff to Sprint 13

Sprint 13 may assume:

- Games run as `playos-game` with no capabilities and `NoNewPrivs` set.
- The control socket is `root:playos-trusted` mode `0660`; games cannot connect.
- Production images are debug-tool-free; development images retain debug tools.
- The Landlock/seccomp sandbox is active on the AMD ROG Ally path.
- Manifest verification and Secure Boot are foundations only, not hard enforcement.

---

## Exit Gate

Game processes are privilege-reduced and cannot access trusted IPC, DRM primary nodes, or other games' data. Production builds ship without debug services. Security restrictions do not break any existing functionality.

*Previous: [Sprint 11.6](Sprint-11.6.md) | Next: [Sprint 13](Sprint-13.md)*
