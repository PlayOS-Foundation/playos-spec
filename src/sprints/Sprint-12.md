# Sprint 12 — Security Hardening

**Goal:** Establish a hardened boundary between the public `playos-platform-api`, trusted `playos-runtime` control paths, and untrusted game processes. Games operate with minimal privileges. Remote debug services are removed from production builds. Secure Boot signing chain is defined.

**Primary Outcome:** A game process cannot access trusted IPC endpoints, cannot open DRM primary nodes, cannot write outside its own save/cache directories, and cannot synthesize reserved system input. Production builds ship without a shell or debug services.

**Prerequisites:** Sprint 11 complete — immutable images and A/B updates working.

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

- **Game identity:** games run as `playos-game` (UID ~1000) with no supplementary groups.
- **Capability set:** games start with no capabilities; `prctl(PR_SET_NO_NEW_PRIVS, 1)` is set before exec.
- **DRM access:** games connect through the Wayland seat; direct `/dev/dri/card*` access is denied.
- **Input boundary:** reserved buttons are intercepted at the libinput/seat layer and never forwarded; the `libplayos` mask is defense-in-depth only.
- **Sandbox mechanism:** a seccomp-BPF syscall allowlist plus a Landlock filesystem allowlist, both applied before game exec.
- **Control socket:** `/run/playos/control.sock` is `root:playos-trusted`, mode `0660`; only `playos-shell` and `playos-overlay` are in `playos-trusted`.
- **Manifest signing:** Ed25519 detached signature alongside `manifest.json`; verification is warn-only this sprint.
- **Secure Boot:** documentation plus development signing keys only; production HSM-backed signing is post-MVP.

---

## Scope

### In Scope

- Game process privilege reduction and credential drop in `playos-init`.
- seccomp syscall allowlist and Landlock filesystem allowlist for games.
- Denial of direct DRM primary-node and raw input-device access.
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
br2-external/board/ally/users-table.txt          # playos-game UID ~1000, no supplementary groups
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
| S12-T1 | Drop game privileges at spawn: UID ~1000, `PR_SET_NO_NEW_PRIVS`, capability drop | `playos-init` | not started | |
| S12-T2 | Apply Landlock allowed/denied path policy to game processes | `playos-init` | not started | |
| S12-T3 | Apply seccomp syscall allowlist to game processes | `playos-init` | not started | |
| S12-T4 | Grant only DRM render-node access; deny privileged KMS/master | `playos-init` | not started | |
| S12-T5 | Enforce reserved-button input isolation end-to-end | `playos-compositor`, `playos-platform-api`, `playos-init` | not started | |
| S12-T6 | Harden `control.sock` trusted-client auth and permission checks | `playos-runtime`, `playos-init` | not started | |
| S12-T7 | Strip debug tools/services from the production image | `playos-refdistro` | not started | |
| S12-T8 | Verify signed game manifests (warn-only in MVP) | `playos-init`, `playos-runtime` | not started | |
| S12-T9 | Document Secure Boot chain; create and rotate dev signing keys in image build | `playos-refdistro`, `playos-spec` | not started | |

Update the **Status** column as work progresses: `not started` → `in progress` → `blocked` or `done`.

### S12-T1 — Drop game privileges at spawn

`playos-init` spawns games as the `playos-game` user (UID ~1000, no supplementary groups), calls `prctl(PR_SET_NO_NEW_PRIVS, 1)` on the game process, and drops all capabilities from the effective and permitted sets before `exec`. The `playos-compositor` keeps `CAP_SYS_ADMIN` (DRM master), but games never inherit it.

**Done when:** from inside a running game, `id` shows `uid=playos-game`, `/proc/self/status` shows `NoNewPrivs: 1`, and `CapEff: 0000000000000000`.

### S12-T2 — Apply Landlock filesystem restrictions

Build a Landlock ruleset granting exactly the paths a game needs and default-deny everything else: `/data/games/<game-id>/` read-only, `/data/saves/<game-id>/` and `/data/cache/<game-id>/` read-write, `/tmp` or `/run/game-<id>/` read-write scratch, and `/run/playos/` execute-only for the Wayland socket. Deny other games' data, `/data/config/`, `/run/playos/control.sock`, `/dev/input/event*`, and `/proc/*/`. If the kernel is older than 5.13, fall back to logging-only enforcement with an alert.

**Done when:** a game process cannot `open()` another game's save directory or read `/data/config/`; the unsupported-kernel fallback logs an alert without blocking launch.

### S12-T3 — Apply seccomp syscall allowlist

Generate a seccomp-BPF allowlist at build time covering the core syscall set (memory, file I/O, AF_UNIX sockets, process, signals, time, `getrandom`, limited `prctl`). Deny `mount`, `umount2`, `init_module`, `finit_module`, `ptrace`, `reboot`, `setuid`, `setgid`, `setcap`, and `open`/`openat` on `/proc/*/mem`, `/dev/dri/card*`, `/dev/input/event*`, and the control socket. Use `libseccomp` to generate and test the filter.

**Done when:** `mount()` from a game process returns `EPERM`, and `open()` of `/proc/*/mem`, `/dev/dri/card*`, `/dev/input/event*`, or the control socket is denied.

### S12-T4 — Restrict DRM node access

`/dev/dri/card*` is owned by the `drm` group; games are not in `drm`. Games reach the GPU through the Wayland seat rather than by opening device nodes directly. Verify the primary node is unopenable from the game identity.

**Done when:** `open("/dev/dri/card0", O_RDWR)` from a game process returns `EACCES`.

### S12-T5 — Enforce reserved-button input isolation

Reserved buttons (`SYSTEM`, `QUICK_MENU`) are intercepted by the compositor at the libinput/seat layer and never forwarded to clients. Games run outside the `input` group, so `/dev/input/event*` is unopenable; Landlock and seccomp also name those paths explicitly. The existing `libplayos` software mask remains as defense-in-depth, not the sole mechanism.

**Done when:** `open("/dev/input/event0", O_RDONLY)` from a game process returns `EACCES`, and `SYSTEM`/`QUICK_MENU` never appear in a game's input stream.

### S12-T6 — Harden control socket

`/run/playos/control.sock` is owned by `root:playos-trusted` with mode `0660`. Only `playos-shell` and `playos-overlay` are in the `playos-trusted` group; game processes are never in that group.

**Done when:** `connect()` to `/run/playos/control.sock` from a `playos-game` process returns `EACCES`.

### S12-T7 — Remove debug tools from production build

The production `defconfig` excludes BusyBox (`/bin/sh`, `/bin/busybox`), the SSH daemon, `gdbserver`/`strace`/`ltrace`, `evtest`/`modetest` and graphics diagnostic tools, and ships with no open listening TCP/UDP sockets. A post-build script asserts their absence, and CI has a production-image lint step that fails if debug artifacts are present. The development image retains all debug tools behind `BR2_PACKAGE_PLAYOS_DEV_TOOLS`.

**Done when:** the production image contains no `busybox`, `gdbserver`, or `strace` and no open sockets; the CI lint step fails if any of these appear.

### S12-T8 — Verify signed game manifests (warn-only)

Define the manifest signing format: an Ed25519 detached signature alongside `manifest.json`. Implement signature verification in `playos-init`, but run it in warn-only mode — log a warning if the signature is missing or invalid, and do not block launch. Hard enforcement is deferred to a later sprint or post-MVP.

**Done when:** an unsigned manifest produces a warning in the log and the game still launches.

### S12-T9 — Document Secure Boot chain and development keys

Document the target signed chain: UEFI Secure Boot signs `BOOTX64.EFI`; `BOOTX64.EFI` signs or contains the kernel; the kernel verifies the initramfs via dm-verity or IMA; A/B update bundles are signed with the PlayOS update key. Generate a self-signed development EFI signing key and add `sbsign`/`pesign` to the build pipeline. Production signing uses an HSM-backed key (post-MVP).

**Done when:** the development EFI key exists under `keys/dev/` and is used by CI builds, `security-model.md` documents the chain, and production HSM signing is explicitly deferred.

---

## Implementation Guidance

**Drop privileges before exec, not after.** The game must never execute with root credentials; perform the credential drop and `PR_SET_NO_NEW_PRIVS` in the child process before `execve`.

**Name the denied paths explicitly.** Both the seccomp `open`/`openat` filter and the Landlock ruleset must name `/dev/input/event*`, `/dev/dri/card*`, `/proc/*/mem`, and the control socket. Do not rely on default-deny alone.

**Treat the `libplayos` mask as defense-in-depth.** The real boundary is the seat plus group permissions plus Landlock plus seccomp; the software bitmask is a convenience, not the enforcement point.

**Landlock fallback must be loud, not silent.** If the kernel is too old, log an alert and continue, but record it as an unresolved hardening gap rather than failing the whole launch path.

**Production lint fails the build.** Debug-artifact absence is enforced in CI, not checked manually.

**Keep Secure Boot scoped this sprint.** Ship documentation and development keys only; do not attempt production key management.

---

## Verification and Evidence

| Evidence | How it is produced |
|---|---|
| Game runs as `playos-game` with no capabilities | `id` and `/proc/self/status` captured inside a test game |
| Primary DRM node denied | Security test binary attempts `open("/dev/dri/card0", O_RDWR)` |
| Raw input device denied | Security test binary attempts `open("/dev/input/event0", O_RDONLY)` |
| Reserved buttons absent | Input-stream dump while `SYSTEM`/`QUICK_MENU` are pressed |
| Control socket denied | Security test binary attempts `connect()` to `control.sock` |
| seccomp active | Security test binary attempts `mount()` and observes `EPERM` |
| Landlock active | Security test binary reads another game's save directory and is denied |
| Production image is debug-free | CI production lint log and post-build script output |
| Manifest warn-only behavior | Launch log for an unsigned manifest |
| Secure Boot chain documented | `security-model.md` plus `keys/dev/` in CI artifacts |

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

*Previous: [Sprint 11](Sprint-11.md) | Next: [Sprint 13](Sprint-13.md)*
