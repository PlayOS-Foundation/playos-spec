# Sprint 11.5 — Pivot-to-Squashfs Boot and A/B Validation

**Goal:** Complete Sprint 11's remaining critical path — make the running system actually boot from the read-only squashfs active slot (instead of the embedded initramfs), and validate the full A/B update/rollback cycle end-to-end.

**Primary Outcome:** On boot, the initramfs mounts the active slot's squashfs image read-only and pivots into it; `touch /usr/test` fails with `EROFS`; a bad slot automatically rolls back after 3 failed boots; the full A/B test matrix passes on QEMU and the ROG Ally.

**Status:** 🟡 Partially validated — T1–T4 landed; host tests + QEMU (pivot + forced rollback) pass. The full 6-case A/B matrix still needs a ROG Ally hardware run.

> **ROG Ally hardware gate:** execute the full 6-case matrix on real hardware only after **[Sprint 11.6](Sprint-11.6.md)** is complete and the ROG Ally is reachable over the network via SSH (USB-C Ethernet + Dropbear). Sprint 11.6 provides the remote access needed to run and capture the matrix directly, instead of reading logs off a USB stick.

**Prerequisites:** Sprint 11 host-side stack landed and committed (boot.json read/write + slot selection, boot-count/rollback logic, `.playosb` bundle format + sign/verify, `ApplyUpdate` IPC, shell update UI, `playos_system_os_version()`).

---

## Why This Sprint Exists

Sprint 11 delivered the A/B update *machinery*, but the runtime still boots entirely from the embedded initramfs. The squashfs image written to the inactive slot is never mounted as `/`, so "immutable root" and automatic rollback are not real end-to-end yet. This sprint closes that gap with the actual boot-path change, then validates it.

---

## Start Condition Checklist

- [x] `src/boot_slot.c` reads/writes `/EFI/playos/boot.json` and selects the active slot.
- [x] Boot-count increment + 3-strike rollback logic exists (ShellReady OR 60-second timer → `mark_good`).
- [x] `.playosb` bundle creation (`scripts/create-update-bundle.sh`) and verification (`src/sha256.c` + `src/update.c`) exist.
- [x] Shell software-update UI and `playos_system_os_version()` (reads active slot from `boot.json`) exist.
- [x] **Open question (resolve first):** does the `playos-refdistro` build produce a *complete bootable squashfs rootfs* (full userspace: shell, compositor, runtime, platform-api, samples, libs, config), or only a partial artifact that the updater writes? This decides whether T1 is "wire the pivot" (small) or "build a full rootfs + minimal initramfs shim" (large). **Resolved:** the `ally` defconfig already builds a complete bootable `rootfs.squashfs` (full userspace present), so this sprint is the small "wire the pivot" path.

---

## Decisions Locked for This Sprint

- **Initramfs role:** becomes a minimal early-boot shim; the real root is the active slot's squashfs image.
- **Boot sequence:** mount ESP read-write → read `boot.json` → select active slot → mount its squashfs read-only → mount `/data` (rw) and `/misc` → `pivot_root`/`switch_root` → `exec` the system init.
- **Read-only guarantee:** squashfs is inherently read-only; no `MS_RDONLY` remount trickery needed. dm-verity remains post-MVP.
- **Slot metadata:** stays in `boot.json` on the ESP (no migration to `misc` this sprint).
- **No repartitioning:** reuse the Sprint 10 5-partition layout unchanged.

---

## Scope

### In Scope

- `playos-refdistro` — full bootable squashfs rootfs (if the open question shows it is partial) + a minimal initramfs shim.
- `playos-init` — `pivot_root`/`switch_root` into the active slot squashfs; wire `boot_slot.c`'s active-slot selection into the mount path; remount `data`/`misc` inside the new root.
- `playos-init` — make 3-strike rollback real end-to-end (a slot that fails to boot actually triggers a switch + reboot).
- Validation matrix (was Sprint 11 S11-T9) on QEMU first, then the ROG Ally.

### Explicitly Out of Scope

- dm-verity (post-MVP)
- Network update download (post-MVP)
- Production HSM update key (post-MVP)
- Delta updates (post-MVP)
- Migrating slot metadata from the ESP to `misc`

---

## Required Repository Changes

| Repo | Required work |
|---|---|
| `playos-refdistro` | Full bootable squashfs rootfs build (if needed) + minimal initramfs shim |
| `playos-init` | `pivot_root`/`switch_root` into the active slot squashfs; slot-selection wiring; real rollback |
| `playos-spec` | This document |

---

## Expected Files and Directories

### `playos-refdistro`

```text
br2-external/board/ally/              # rootfs-as-squashfs build changes (if the open question shows a partial rootfs)
br2-external/board/ally/initramfs/    # minimal early-boot shim (mount squashfs -> pivot_root)
```

### `playos-init`

```text
src/boot_slot.c                       # already implements selection; wire into the mount path
src/main.c                            # replace embedded-rootfs boot with pivot into the active slot
src/mount.c                           # mount active slot squashfs + remount data/misc in the new root
```

---

## Agent Task Breakdown

### Task Status Grid

| Task ID | Task | Primary repo | Status | Notes / evidence |
|---|---|---|---|---|
| S11.5-T1 | Confirm/produce a complete bootable squashfs rootfs + minimal initramfs shim | `playos-refdistro` | done | Full userspace confirmed in `output/ally/images/rootfs.squashfs`; added `/EFI` mountpoint to rootfs-overlay. |
| S11.5-T2 | `pivot_root`/`switch_root` into the active slot squashfs | `playos-init` | done | `playos_pivot_to_active_slot()` added in `src/mount.c`; switch_root idiom (MS_MOVE + chroot + exec /init). |
| S11.5-T3 | Wire `boot_slot.c` active-slot selection into the mount path | `playos-init` | done | `boot_slot_read()` selects `playos-a`/`playos-b`; called in `main.c` after ESP/boot-slot block. |
| S11.5-T4 | Make 3-strike rollback real end-to-end | `playos-init` | done | Host `test_boot_slot` covers boot-count/3-strike rollback; QEMU Scenario B proves forced rollback (boot.json flipped to slot b, slot a marked bad). |
| S11.5-T5 | A/B update + rollback validation matrix (was S11-T9) | `playos-refdistro` | in progress | Automated subset passes (host `test_boot_slot` 2/2 + QEMU Scenario A/B); **fresh install → 5-partition layout → NVMe boot validated 2026-08-19** (S10-T8 re-install fix). Full 6-case matrix on ROG Ally still pending (real apply/reboot/mark-good timing, `/data` survival on real NVMe, live version API per slot) — **blocked on [Sprint 11.6](Sprint-11.6.md)** SSH/network reachability for remote execution. |

---

### S11.5-T1 — Confirm/produce a complete bootable squashfs rootfs + minimal initramfs shim

**Finding (to confirm before any code):** the system currently boots from the embedded initramfs. It is unknown whether the refdistro build already produces a complete squashfs rootfs or only a partial artifact for the updater.

**Steps:**

1. Inspect the refdistro build output: locate the squashfs artifact, list its top-level entries, and confirm whether it contains a complete userspace (shell, compositor, runtime, platform-api, samples, `/lib`, `/etc`).
2. If complete: proceed to T2 — only the initramfs pivot path needs wiring.
3. If partial: build the full rootfs as squashfs and reduce the initramfs to a minimal shim (mount ESP → read `boot.json` → mount active squashfs → `pivot_root` → `exec /sbin/init`).

**Done when:** the build produces a squashfs that contains the full system userspace, and the initramfs no longer carries the whole system.

---

### S11.5-T2 — `pivot_root`/`switch_root` into the active slot squashfs

**Steps:**

1. In `playos-init/src/main.c`, after the early mount of the ESP and `boot.json` read, mount the active slot partition (by GPT label `playos-a`/`playos-b`) read-only at a staging path.
2. Mount `/data` (label `playos-data`) read-write and `/misc` under the staged new root.
3. `pivot_root` (or `switch_root` if a minimal initramfs shim is used) into the squashfs, then `exec` the real init.
4. Log the transition: `playos-init: pivoted to active slot <a|b> (squashfs, read-only)`.

**Done when:** QEMU boots and `cat /proc/mounts` shows `/` on `squashfs` with `ro`; `touch /usr/test` fails with `EROFS`.

---

### S11.5-T3 — Wire `boot_slot.c` active-slot selection into the mount path

**Steps:**

1. Reuse the existing `boot_slot.c` slot-selection result to choose `playos-a` vs `playos-b` at mount time.
2. Confirm a missing/corrupt `boot.json` falls back to slot A and recreates the file (behavior already specified in Sprint 11).
3. Keep `boot_slot.c` as the single source of truth — do not duplicate slot-selection logic in `main.c`.

**Done when:** changing `active_slot` in `boot.json` causes the next boot to mount the other slot.

---

### S11.5-T4 — Make 3-strike rollback real end-to-end

**Steps:**

1. Verify the existing rollback path fires when the new root fails to boot: `boot_count >= 3 && health != "good"` → mark `bad`, switch `active_slot`, reboot.
2. Ensure the boot-count increment and `mark_good` (ShellReady/60s) survive the pivot — they must operate on the ESP, which stays mounted across the pivot.
3. Test that a deliberately broken slot (bad initramfs or missing squashfs) rolls back to the previous slot after 3 attempts.

**Done when:** corrupting the active slot causes 3 failed boots and an automatic switch to the previous slot.

---

### S11.5-T5 — A/B update + rollback validation matrix

Full test matrix (QEMU first, then ROG Ally):

1. Fresh install → 5-partition layout → `boot.json` shows slot A good.
2. Apply valid bundle → `boot.json` shows slot B pending → reboot → slot B active → 60s → slot B good.
3. Corrupt slot B → reboot 3× → rollback to slot A.
4. `/data` content (game files, saves) survives update and rollback unchanged.
5. Invalid-signature bundle → rejected before any partition write.
6. `playos_system_os_version()` returns the correct version for each slot.

**Done when:** all 6 cases pass with log evidence.

---

## Implementation Guidance

### Resolve T1 before coding T2–T4

T1 determines whether this sprint is a small boot-path wiring change or a larger rootfs-build effort. Do the refdistro build inspection first and report the finding before writing any `pivot_root` code.

### Keep `boot_slot.c` the single source of truth

Slot selection, boot counting, and rollback all live in `boot_slot.c`. `main.c` should consume its result, not re-derive it.

### Atomic commits

```
S11.5-T1: build full bootable squashfs rootfs + minimal initramfs shim
S11.5-T2: pivot into active slot squashfs
S11.5-T3: wire active-slot selection into boot mount path
S11.5-T4: make 3-strike rollback real end-to-end
S11.5-T5: A/B validation matrix + evidence
```

---

## Verification and Evidence

| Evidence | How it is produced | Current state |
|---|---|---|
| Read-only root | `cat /proc/mounts` shows `/` on `squashfs` `ro`; `touch /usr/test` → `EROFS` | ⚠️ Not yet asserted by the QEMU harness (pivot is asserted; ro/EROFS check not captured) |
| Slot selection | changing `active_slot` in `boot.json` changes the mounted slot | ✅ QEMU Scenario A pivots to slot `a` read from `boot.json`; Scenario B writes `active_slot: "b"` |
| Rollback | 3-strike test log showing automatic slot switch | ✅ QEMU Scenario B flips to `b` and marks `a` `bad`; host `test_boot_slot` covers boot-count logic |
| Update apply/verify | host `test_boot_slot` valid / bad-signature / bad-magic cases | ✅ `ctest` 2/2 pass |
| Data survival | file hashes before/after update match | ⚠️ Pending ROG Ally — not covered by the QEMU harness |
| Signature rejection | log showing bundle rejected before any write | ✅ host `test_boot_slot` bad-signature case |
| Version API | `playos_system_os_version()` per slot | ⚠️ Code inspection only; hardware per-slot run pending |

---

## Acceptance Criteria

- [x] System boots from the read-only squashfs active slot (not the embedded initramfs) — QEMU Scenario A
- [ ] `touch /usr/test` fails with `EROFS` — squashfs ro is by construction, but the current QEMU harness does not assert this
- [x] Active-slot selection in `boot.json` drives which slot is mounted — QEMU Scenario A reads `active_slot: "a"`
- [x] A valid bundle applied to the inactive slot does not affect the running system — host `test_boot_slot` apply path (by design, apply writes only to the inactive slot)
- [ ] After reboot, the system boots from the newly updated slot — full apply→reboot→active cycle not yet run on QEMU or hardware
- [ ] Successful boot marks the new slot `health = "good"` after 60 seconds — logic covered by host test; real 60s timer not yet exercised
- [x] 3 consecutive failed boots trigger rollback to the previous slot — QEMU Scenario B + host `test_boot_slot`
- [ ] `/data` content survives update and rollback unchanged — not yet exercised on QEMU or hardware
- [x] Invalid-signature bundle is rejected before any partition write — host `test_boot_slot` bad-signature case
- [x] QEMU validation passes for the bootable subset — Scenario A (pivot) + Scenario B (forced rollback)
- [ ] ROG Ally passes the full 6-case matrix — pending [Sprint 11.6](Sprint-11.6.md) SSH/network reachability

---

## Handoff to Sprint 12

Sprint 12 (Security Hardening) may assume:

- The system image is immutable at runtime (read-only squashfs root).
- Signed A/B updates with automatic rollback are functional on the automated/QEMU subset; the full hardware matrix is pending Sprint 11.5 closure, executed over SSH after [Sprint 11.6](Sprint-11.6.md) is complete.
- Games and user data live on `/data` and are unaffected by updates and rollback.

---

## Exit Gate

The system boots from the read-only squashfs active slot, A/B updates can be applied and automatically rolled back on failure, and the full validation matrix passes on QEMU and the ROG Ally (the Ally portion executed over SSH once [Sprint 11.6](Sprint-11.6.md) is complete).

*Previous: [Sprint 11](Sprint-11.md) | Next: [Sprint 12](Sprint-12.md)*
