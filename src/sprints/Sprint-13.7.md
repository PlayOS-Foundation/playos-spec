# Sprint 13.7 — Live-USB / Installer Image Consolidation

**Goal:** Collapse the separate Live-USB and Installer image families into **two consolidated images per target** — a **dev** image and a **prod** image — each of which boots live to the PlayOS shell and, from a Settings action, installs PlayOS onto the internal disk via a runtime installer handoff. This eliminates the duplicated installer defconfigs, kernel configs, and generation scripts that Sprint 13's Intel expansion introduced, while preserving the dev/prod SSH split.

**Primary Outcome:** Two images per target boot the full shell as a live session and install via **Settings → System → Install PlayOS to internal disk** (with confirmation). The action tears down the live `/data` and `/EFI` mounts and hands control to the same trusted installer that previously shipped as a separate image; on completion the device reboots into the freshly installed OS.

- **Dev image** (`playos-ally-dev-usb.img` / `playos-intel-dev-usb.img`): SSH/DropBear present on **both** the live session and the installed system.
- **Prod image** (`playos-ally-prod-usb.img` / `playos-intel-prod-usb.img`): no SSH/DropBear on the live session **or** the installed system.

The dedicated installer defconfigs and `gen-installer-usb-image.sh` are deleted. The only dev/prod difference is SSH/DropBear presence.

**Status:** 🟢 In progress — Phases 1-3 (T1-T6) implemented and host-tested; Phase 4 (T7 QEMU + on-device validation) pending. Design locked on **Option B (runtime installer handoff)**, two-flavor (dev + prod).

**Prerequisites:** Sprint 13 complete (Intel expansion, two live targets); Sprint 10 installer (`src/playos-installer`) verified; `playos-init` supervision + `playos_supervisor_spawn_installer` path present; `playos-runtime` trusted-control IPC in place.

---

## Why This Sprint Exists

Since Sprint 10, PlayOS has shipped two image *families*: a Live-USB image that boots to the shell, and an Installer image that boots straight into `playos-installer`. Sprint 13's Intel expansion doubled that to four images and duplicated the installer kernel configs (`linux-installer.config`, `linux-installer-fragment.cfg`) and the `playos_ally_installer_defconfig` / `playos_intel_installer_defconfig` defconfigs. Every kernel, package, and security change now has to land in two places, and the two-kernel staging logic in `gen-installer-usb-image.sh` (ESP `BOOTX64.EFI` = installer kernel with `playos.mode=install`; `playos-a/BOOTX64.EFI` = the installed kernel written verbatim to the target ESP, plus `rootfs.squashfs`) is easy to get wrong.

Option B removes the duplication by making the **live image itself** the installer: the shell launches the trusted installer at runtime through `playos-init`'s supervisor, reusing the existing `playos_supervisor_spawn_installer` path instead of gating on the boot-time `playos.mode=install` cmdline flag. Each flavor becomes one self-contained image that is both a demo (boots to shell) and an installer (Settings action), and the maintainers keep a single defconfig + single kernel config per flavor — while the dev/prod SSH split stays intact.

---

## Start Condition Checklist

- [ ] Sprint 13 complete — Intel expansion landed with `playos_intel_pc_defconfig` + `playos_intel_installer_defconfig`.
- [ ] Sprint 10 installer (`src/playos-installer/main.c`, 8-step state machine) installs to internal disk from removable media and passes QEMU + Ally validation.
- [ ] `playos-init/src/main.c:257-260` already dispatches `s->install_mode` → `playos_supervisor_spawn_installer(s)`; supervisor has `installer_pid` / `installer_restarts` / `installer_should_restart` machinery.
- [ ] `playos-init/src/ipc_handler.c` already handles `SHUTDOWN`/`REBOOT` (calls `playos_shutdown`); `playos-runtime` already wraps `launch_game`, `shutdown`, `reboot` via `trusted_control.c`.
- [ ] Shell Settings already has the reboot/shutdown confirm pattern at `playos-shell/src/screen_settings.c:251-279` under `#ifdef PLAYOS_TRUSTED_IPC`.
- [ ] All live defconfigs set `BR2_TARGET_ROOTFS_SQUASHFS=y` and `BR2_LINUX_KERNEL_INITRAMFS_SOURCE="$(BINARIES_DIR)/rootfs.cpio"`, so a live build already emits `rootfs.squashfs`.
- [ ] Dev defconfig has Dropbear/SSH (`output/ally`, `output/intel`); prod defconfig is stripped of Dropbear/BusyBox (`output/ally-production`).

---

## Decisions Locked for This Sprint

- **Option B — runtime installer handoff.** The shell triggers installation at runtime; there is **no** reboot-into-installer-mode and **no** separate installer defconfig. This is the locked design; Option A (keep two images) is rejected.
- **Two consolidated images per target — dev + prod.** Each image boots live to shell *and* carries its own install payload (`rootfs.squashfs` + the flavor's normal `BOOTX64.EFI`) on the `playos-a` partition for the installer to discover. Exact final filenames are a T5 detail; the proposed names are `playos-ally-dev-usb.img` / `playos-ally-prod-usb.img` (and `-intel-` equivalents).
- **Dev/Prod split = SSH/Dropbear presence only.** The dev image has Dropbear/SSH on the live session and the installed system; the prod image has neither on live nor installed. The installer and the Settings install action are present in **both** live images (otherwise a prod image could not install). Nothing else differs between the flavors.
- **Handoff path (review-corrected order).** `StartInstaller` IPC → `playos-init` sets a runtime-installer flag, **stops the current shell + overlay first** (closes their `/data` log fds), then unmounts `/data` → `/EFI`, then calls `playos_supervisor_spawn_installer(s)`. Unmounting *before* stopping the shell would hit EBUSY because init redirects shell/overlay stderr to `/data/log/*`. If an unmount still fails, init **respawns shell + overlay** and returns an ERROR ack, leaving the live session running. This reuses the existing spawn path, decoupled from `s->install_mode`.
- **Post-install = reboot.** There is no "back to shell" mid-session; the installer completes and the device reboots into the installed OS. Unmount failures abort cleanly and keep the shell running.
- **Payload stays on `playos-a`, flavor-matched.** The installer discovers its payload independently via `find_and_mount_payload()` (mount `playos-a` by label, check `rootfs.squashfs` + `BOOTX64.EFI`); it does not depend on init's mounts. **Dev stages the dev rootfs** (Dropbear/SSH present); **prod stages the prod rootfs** (no Dropbear/BusyBox). This is what preserves the SSH split across an install.
- **Install action gated on payload discoverability.** The Settings install action is shown only when the `playos-a` payload partition (with `rootfs.squashfs` + `BOOTX64.EFI`) is discoverable. An installed system has no such partition, so it never offers "Install PlayOS" — re-install is always performed by booting the matching USB image again. This holds for both dev and prod.
- **Kernel artifacts are the same per flavor (review correction).** All live defconfigs embed `rootfs.cpio` into `bzImage`; the installed system boots from that embedded rootfs exactly like the live session. The staged `playos-a/BOOTX64.EFI` is therefore the **same `bzImage` as the live ESP kernel** — do NOT build a second "no-initramfs" kernel. `playos-a` carries `rootfs.squashfs` (payload slot) + the flavor's `bzImage` (ESP kernel).
- **Known trade-off (documented, default accepted).** Because each flavor is one build, the installed rootfs is the same artifact as the live rootfs — so the **installed prod** system still ships the inert `playos-installer` binary. It is harmless: there is no payload partition and the install action is gated off. If a truly installer-free installed-prod rootfs is later desired, stage a second, installer-stripped squashfs per flavor (optional follow-up, out of scope here).
- **SSH-key seed follows the dev flavor only.** The dev image continues to seed `data/ssh/authorized_keys` (so a fresh dev install is immediately SSH-able — Sprint 11.6 dependency). The prod image does **not** seed an SSH key (no Dropbear present).
- **Headless QEMU automation preserved.** Keep a scriptable path so CI can drive the install without a controller (e.g. a cmdline token that maps to the same supervisor transition, or a StartInstaller IPC sent over the existing trusted socket). Exact mechanism chosen at implementation time; it must not reintroduce a separate installer defconfig.
- **Register-shell conflict resolved implicitly.** Because init terminates shell + overlay *before* spawning the installer, exactly one "shell" surface exists at any instant — no compositor or `register_shell` changes are needed.

---

## Review Findings (2026-08-30, implementation plan)

Review of Sprint-13.7 against the current code verified all start-condition
prerequisites and found four corrections that are now baked into the decisions
above:

1. **Installed kernel = the flavor's normal `bzImage`** (which embeds the full
   rootfs via `rootfs.cpio`). The original "no embedded initramfs" claim did
   not match any live defconfig; no second kernel build is needed.
2. **Handoff order: stop shell+overlay → unmount `/data` → `/EFI` → spawn
   installer.** The original order (unmount first) would EBUSY on the shell's
   `/data/log/shell-stderr.log` fd. On unmount failure, respawn shell+overlay
   and return an ERROR ack.
3. **Runtime installer exit → reboot, not restart.** The supervisor's existing
   exit path restarts the installer; a runtime-mode flag makes it reboot after
   a Settings-triggered install. Boot-time `playos.mode=install` keeps the
   restart behavior for QEMU automation.
4. **Shell payload gating = shell-side check** of `/dev/disk/by-label/playos-a`
   (mount read-only, stat `rootfs.squashfs` + `BOOTX64.EFI`). No IPC extension
   beyond `StartInstaller` is required for T3.

Also confirmed: `playos_supervisor_spawn_installer()` is already public and
decoupled; the installer package already depends on `util-linux dosfstools
e2fsprogs efibootmgr`; `find_and_mount_payload()` already mounts `playos-a` by
label and validates both payload files; the live gen scripts already create an
empty `playos-a` partition (populating it is T5).

---

## Scope

### In Scope

- `playos-runtime` — add `PLAYOS_IPC_TYPE_START_INSTALLER "StartInstaller"` and `playos_trusted_start_installer(fd)`; mirror the constant in `playos-init/ipc/ipc.h`.
- `playos-init` — handle `StartInstaller` in `ipc_handler.c`: stop shell + overlay, unmount `/data` then `/EFI`, spawn the installer via the supervisor; on installer exit in runtime mode, reboot.
- `playos-shell` — Settings **Install PlayOS to internal disk** action + confirmation, calling `playos_trusted_start_installer(-1)` (mirrors reboot/shutdown), gated on payload discoverability; update `playos-shell/AGENTS.md` trusted-ops list.
- `playos-refdistro` — merge `BR2_PACKAGE_PLAYOS_INSTALLER` and installer tools (`blockdev`, `fdisk`, `mkfs.fat`, `mkfs.ext2|4`, `efibootmgr`) into **both** the dev and prod defconfigs (ally + intel).
- `playos-refdistro` — extend `gen-ally-usb-image.sh` and `gen-intel-usb-image.sh` (or their dev/prod variants) to stage the **flavor-matched** `rootfs.squashfs` + normal `BOOTX64.EFI` on `playos-a`; keep the SSH-key seed for the dev image only.
- `playos-refdistro` — delete `playos_ally_installer_defconfig`, `playos_intel_installer_defconfig`, `playos_installer_qemu_defconfig`, `gen-installer-usb-image.sh`, `linux-installer.config`, `linux-installer-fragment.cfg`, and the installer Makefile targets.
- `playos-spec` — this document; `SUMMARY.md` + `roadmap.md` wiring.

### Explicitly Out of Scope

- Re-architecting the installer state machine or adding progress/telemetry beyond what already exists.
- A shell-based partition picker or disk-selection UI (the installer keeps auto-selecting the first fixed internal disk).
- Resumable / non-destructive upgrade installs — out of scope; installs remain destructive to the target disk.
- Stripping the `playos-installer` binary from the installed prod rootfs (optional follow-up — see the locked trade-off).
- Production Dropbear/BusyBox stripping itself — that is Sprint 12's concern; this sprint only controls whether `playos-installer` and the install action are present.
- OTA updates, `playos-update`, or `playos-tools` — post-MVP.
- Renumbering downstream sprints (14–22) — this slots in as 13.7 without touching them.

---

## Required Repository Changes

| Repo | Required work |
|---|---|
| `playos-runtime` | Add `StartInstaller` IPC constant + `playos_trusted_start_installer(fd)` wrapper |
| `playos-init` | Handle `StartInstaller` → teardown `/data`+`/EFI` → supervisor swap to installer → reboot on exit |
| `playos-shell` | Settings install action + confirm (`playos_trusted_start_installer(-1)`), gated on payload; update `AGENTS.md` |
| `playos-refdistro` | Merge installer package/tools into dev + prod defconfigs; stage flavor-matched payload in gen scripts; delete installer defconfigs + scripts + Makefile targets |
| `playos-spec` | This sprint; `SUMMARY.md` + `roadmap.md` links |

---

## Expected Files and Directories

### `playos-runtime`

```text
include/playos-runtime/trusted_control.h   # playos_trusted_start_installer(int fd)
src/trusted_control.c                        # wrapper implementation
# IPC string macro mirrored in playos-init/ipc/ipc.h
```

### `playos-init`

```text
ipc/ipc.h                 # PLAYOS_IPC_TYPE_START_INSTALLER "StartInstaller" (+ ACK/ERROR variants)
src/ipc_handler.c         # StartInstaller handler: unmount /data + /EFI, then signal supervisor
src/supervisor.c          # playos_supervisor_start_installer(s) reusing spawn_installer, decoupled from s->install_mode
```

### `playos-shell`

```text
src/screen_settings.c     # "Install PlayOS to internal disk" action + confirmation (mirrors reboot/shutdown), gated on payload
AGENTS.md                 # add start_installer to the trusted-ops list
```

### `playos-refdistro`

```text
br2-external/configs/playos_ally_defconfig            # + BR2_PACKAGE_PLAYOS_INSTALLER + tools (dev)
br2-external/configs/playos_ally_production_defconfig # + BR2_PACKAGE_PLAYOS_INSTALLER + tools (prod, still no Dropbear/BusyBox)
br2-external/configs/playos_intel_pc_defconfig        # + BR2_PACKAGE_PLAYOS_INSTALLER + tools (dev)
# (Intel prod defconfig if/when it exists)            # + BR2_PACKAGE_PLAYOS_INSTALLER + tools (prod)
scripts/gen-ally-usb-image.sh                         # stage flavor-matched rootfs.squashfs + normal BOOTX64.EFI on playos-a
scripts/gen-intel-usb-image.sh                        # same
# deleted:
#   br2-external/configs/playos_ally_installer_defconfig
#   br2-external/configs/playos_intel_installer_defconfig
#   br2-external/configs/playos_installer_qemu_defconfig
#   scripts/gen-installer-usb-image.sh
#   br2-external/board/ally/linux-installer.config
#   br2-external/board/ally/linux-installer-fragment.cfg
#   Makefile installer-* targets
```

---

## Agent Task Breakdown

### Task Status Grid

| Task ID | Task | Primary repo | Status | Notes / evidence |
|---|---|---|---|---|
| S13.7-T1 | `StartInstaller` IPC + trusted wrapper | `playos-runtime`, `playos-init` | done | init `18679c8`, runtime `85acbb9`; tests green |
| S13.7-T2 | `playos-init` runtime installer handoff | `playos-init` | done (code) | init `18679c8`; on-device validation pending in T7 |
| S13.7-T3 | Shell Settings install action | `playos-shell` | done (code) | shell `d1729a4`; on-device validation pending in T7 |
| S13.7-T4 | Merge installer into dev + prod defconfigs | `playos-refdistro` | done | refdistro `18d95a3` |
| S13.7-T5 | Stage flavor-matched payload in gen scripts | `playos-refdistro` | done | refdistro `18d95a3`; gen scripts accept `[image-name] [flavor]` |
| S13.7-T6 | Delete installer-only artifacts | `playos-refdistro` | done | refdistro `18d95a3`; no refs remain |
| S13.7-T7 | QEMU + Ally validation | `playos-refdistro` | done | QEMU headless runtime-install PASSED; on-device Ally validated: live USB boots without slot pivot (ESP marker), Settings install action works, reboot into installed NVMe, dev SSH key seeded and SSH reachable |

Update the **Status** column as work progresses: `not started` → `in progress` → `blocked` or `done`.

### S13.7-T1 — `StartInstaller` IPC + trusted wrapper

- Add `PLAYOS_IPC_TYPE_START_INSTALLER "StartInstaller"` to `playos-init/ipc/ipc.h` (and the ACK/ERROR variants) and mirror it wherever `playos-runtime` keeps its copy of the string macros (the constants are shared string macros, not a binary ABI).
- Add `playos_trusted_start_installer(int fd)` to `playos-runtime` `trusted_control.h` / `trusted_control.c`, following the existing `playos_trusted_reboot` / `playos_trusted_shutdown` pattern: open a fresh connect → send → recv ack → close. Do **not** pre-open a connection (the shell documents a single-connection caveat).

**Done when:** the new message type is recognized by init's IPC handler and the wrapper sends/receives it without leaking a connection.

### S13.7-T2 — `playos-init` runtime installer handoff

- In `ipc_handler.c`, handle `StartInstaller`:
  1. If `s->install_mode` (boot-time flag) is set, this is a no-op — the installer is already running.
  2. Otherwise set `s->installer_runtime_mode = true` (new field), then ask the supervisor to **stop the current shell + overlay** (closing their `/data/log/*` fds).
  3. Unmount `/data` (`playos-data`) then `/EFI` (`ESP`), logging each result. If either unmount fails (busy), **respawn shell + overlay**, clear the runtime flag, and return an ERROR ack (live session keeps running).
  4. Call `playos_supervisor_spawn_installer(s)`, reusing the existing `installer_pid` / `installer_restarts` machinery.
- In the supervisor's installer-exit path: if `installer_runtime_mode` is set, reboot into the installed OS instead of restarting the installer (do not re-mount `/data`/`/EFI` mid-session). Boot-time `playos.mode=install` keeps the existing restart behavior for QEMU automation.
- Decouple this path from `s->install_mode`: the transition is driven by the IPC message, not the cmdline flag, while the boot-time flag keeps working for QEMU automation.

**Done when:** sending `StartInstaller` from a live boot stops shell+overlay, tears down the mounts, spawns the installer under supervisor, and reboots on completion; a busy-mount failure returns an error and the shell is respawned.

### S13.7-T3 — Shell Settings install action

- Add a **System → Install PlayOS to internal disk** action in `screen_settings.c`, mirroring the reboot/shutdown confirm pattern at lines 251-279 (`#ifdef PLAYOS_TRUSTED_IPC` → `playos_trusted_reboot(-1)` / `playos_trusted_shutdown(-1)`).
- Gate the action on payload discoverability: only render it when the `playos-a` partition with `rootfs.squashfs` + `BOOTX64.EFI` is present, so an installed system never shows it. Implementation (review decision): shell checks `/dev/disk/by-label/playos-a`, mounts it read-only (e.g. at `/mnt/playos-payload-check`), stats `rootfs.squashfs` + `BOOTX64.EFI`, then unmounts — no IPC extension beyond `StartInstaller`.
- On confirm, call `playos_trusted_start_installer(-1)`. Show an "Installing…" state and rely on the init handoff for the actual transition.
- Update `playos-shell/AGENTS.md` (the trusted-ops list around lines 120-127) to include `start_installer`.

**Done when:** the Settings action appears on the live image, is absent on an installed system, presents a confirmation dialog, and on confirm triggers the handoff (verified in T7); the AGENTS.md list is accurate.

### S13.7-T4 — Merge installer into dev + prod defconfigs

- Add `BR2_PACKAGE_PLAYOS_INSTALLER=y` and the installer's runtime tools (`blockdev`, `fdisk`, `mkfs.fat`, `mkfs.ext2`/`mkfs.ext4`, `efibootmgr`) to `playos_ally_defconfig`, `playos_ally_production_defconfig`, and `playos_intel_pc_defconfig` (plus the Intel prod defconfig if it exists).
- Keep the dev/prod SSH split unchanged: dev defconfigs keep Dropbear/SSH; prod defconfigs keep them **out** (no Dropbear/BusyBox). Only the installer package/tools are added to both.
- Confirm no package/feature conflict arises from enabling these in the live (dev and prod) image alongside the existing shell/overlay/audio/input packages.

**Done when:** the built dev and prod rootfs's each contain the `playos-installer` binary and its required tools; dev has Dropbear/SSH; prod does not.

### S13.7-T5 — Stage flavor-matched payload in gen scripts

- Extend `scripts/gen-ally-usb-image.sh` and `scripts/gen-intel-usb-image.sh` (or their dev/prod variants) to:
  - Mount the existing empty `playos-a` partition on the USB image.
  - Copy the **flavor-matched** `rootfs.squashfs` + `bzImage` (the same kernel that goes on the live ESP — review correction; no second kernel) onto `playos-a`: dev image stages the dev rootfs/kernel; prod image stages the prod rootfs/kernel.
  - Seed `data/ssh/authorized_keys` for the **dev image only** (no Dropbear on prod, so no key seed).
- Keep the live ESP `EFI/BOOT/BOOTX64.EFI` as-is; the staged `playos-a/BOOTX64.EFI` is the same `bzImage` artifact.

**Done when:** a dev image's `playos-a` carries the dev `rootfs.squashfs` + dev normal `BOOTX64.EFI`, and a prod image's `playos-a` carries the prod artifacts, discoverable by the installer's `find_and_mount_payload()`; the SSH-key seed is present on dev and absent on prod.

### S13.7-T6 — Delete installer-only artifacts

- Delete `playos_ally_installer_defconfig`, `playos_intel_installer_defconfig`, `playos_installer_qemu_defconfig`.
- Delete `scripts/gen-installer-usb-image.sh` and the `linux-installer.config` / `linux-installer-fragment.cfg` files.
- Remove/retire the `installer-*` Makefile targets in `playos-refdistro/Makefile` (verify exact target names first — they are referenced by the deleted gen script).
- Preserve the headless QEMU automation via the mechanism chosen in Decisions (cmdline token → same supervisor transition, or a scripted StartInstaller IPC), without a separate installer defconfig.

**Done when:** `grep -ri "installer"` in `br2-external/configs/` and `scripts/` returns no references to the deleted defconfigs/script; the headless automation still installs in QEMU.

### S13.7-T7 — QEMU + Ally validation

- QEMU: boot each consolidated image → reach the shell → trigger the install action (or the headless token) → installer completes → reboot lands on the installed OS; confirm `playos.install.auto`-equivalent automation still works.
- Ally: flash `playos-ally-dev-usb.img` → boots live to shell with SSH → installs → reboot boots from NVMe into a dev system that is still SSH-able. Repeat with `playos-ally-prod-usb.img` → live boot has **no** SSH, installs, and the installed system has **no** SSH and **no** install action.
- Confirm the prod image has no Dropbear/BusyBox on live or installed, and no install action appears after internal boot.

**Done when:** both flavors go from live boot to installed OS via the runtime handoff (QEMU + Ally), with log evidence; dev stays SSH-able; prod has no SSH and no re-install surface.

---

## Implementation Guidance

**Reuse the supervisor, don't fork it.** `playos_supervisor_spawn_installer(s)` already exists and is what `main.c` calls in `s->install_mode`. The new IPC path should converge on the same function; the only new work is the teardown (unmount `/data` + `/EFI`) and the trigger.

**Teardown order matters.** Unmount `/data` before `/EFI`, because the shell's overlay may still reference files under `/data`. Abort cleanly on any unmount failure and return an ERROR ack — never leave the shell half-torn-down.

**Preserve the two-kernel split, per flavor.** The live ESP kernel and the staged `playos-a` kernel are different artifacts (embedded initramfs vs. normal). Do not collapse them; the installer still writes the staged normal kernel verbatim to the target ESP.

**The SSH split is the whole point.** Dev stages the dev rootfs (SSH/DropBear on live + installed); prod stages the prod rootfs (no SSH/DropBear anywhere). Do not accidentally stage the dev rootfs into a prod image or vice versa.

**SSH-key seed follows dev only.** Seed `data/ssh/authorized_keys` on the dev image; skip it on prod (no Dropbear present). A fresh dev install must remain immediately SSH-able (Sprint 11.6 dependency).

**Keep it destructive and simple.** No partition picker, no non-destructive upgrade, no resumable install. The installer keeps auto-selecting the first fixed internal disk.

**Gate the install action.** Show "Install PlayOS" only when the `playos-a` payload is discoverable; an installed system never offers re-install. This is the mechanism that keeps both flavors from self-reinstalling after internal boot.

**Accepted trade-off.** Installed prod still carries the inert `playos-installer` binary (one build per flavor). Harmless without a payload partition; stripping it is a documented optional follow-up.

---

## Verification and Evidence

| Evidence | How it is produced |
|---|---|
| Two images per target | `playos-ally-dev-usb.img` / `playos-ally-prod-usb.img` (+ intel) exist; no `gen-installer-usb-image.sh` or installer defconfigs remain |
| Payload staged, flavor-matched | `playos-a` partition carries the matching `rootfs.squashfs` + normal `BOOTX64.EFI` |
| Installer present in live rootfs | `which playos-installer` in the live shell session (dev and prod) |
| Dev/prod SSH split | dev live + dev installed reachable over SSH; prod live + prod installed are not |
| Install action gated | "Install PlayOS" shows on live, absent after internal boot |
| Handoff works | `StartInstaller` log shows `/data`+`/EFI` unmount → installer spawn under supervisor |
| Install completes | installer log shows 8-step success; device reboots into installed OS |
| SSH seed (dev only) | fresh dev install is SSH-able after first boot; prod image has no seeded key |
| Headless automation | QEMU installs via the scripted path without manual input |

---

## Acceptance Criteria

- [ ] A dev image and a prod image per target each boot live to the shell and install to internal disk from Settings
- [ ] Dev image has SSH/Dropbear on live **and** installed; prod image has neither on live **or** installed
- [ ] The Settings install action presents a confirmation, is gated on payload discoverability, and triggers the runtime handoff
- [ ] `playos-init` unmounts `/data` then `/EFI` and spawns the installer via the supervisor on `StartInstaller`
- [ ] A busy-mount failure aborts cleanly and keeps the shell running
- [ ] On installer completion the device reboots into the installed OS
- [ ] Each live image's `playos-a` partition carries its flavor-matched `rootfs.squashfs` + normal `BOOTX64.EFI`
- [ ] The installer defconfigs, `gen-installer-usb-image.sh`, `linux-installer.*`, and installer Makefile targets are removed
- [ ] Headless QEMU installation still works
- [ ] The SSH-key seed is preserved on dev and omitted on prod
- [ ] Installed systems (dev and prod) show no re-install action

---

## Handoff to Sprint 14

Sprint 14 (Production Readiness) may assume:

- There are exactly **two** image artifacts per target for the preview release — a dev image (SSH/DropBear) and a prod image (clean) — with no separate installer image.
- The installer + Settings install action live in both live images; the install action is gated off on installed systems.
- The prod image carries no Dropbear/BusyBox on live or installed; the dev/prod difference is SSH/DropBear presence only.
- The runtime installer handoff is the only supported install path; boot-time `playos.mode=install` survives solely as a QEMU automation convenience.

---

## Exit Gate

Two consolidated images per target (dev + prod) each boot live to the shell and install PlayOS onto the internal disk from a Settings action via `playos-init`'s runtime installer handoff, the dev/prod SSH split is preserved across live and installed, the installer-only artifacts are deleted, headless QEMU automation still installs, and installed systems show no re-install surface.

*Previous: [Sprint 13.6](Sprint-13.6.md) | Next: [Sprint 14](Sprint-14.md)*
