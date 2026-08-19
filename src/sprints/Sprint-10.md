# Sprint 10 — Installer and Internal-Disk Deployment

**Goal:** Build a tested, user-confirmed installation path from a USB boot image to the ROG Ally internal NVMe SSD. After installation, the Ally boots PlayOS from the internal disk without a USB drive.

**Primary Outcome:** A user can boot the PlayOS installer from USB, confirm the target disk, wait for installation, remove the USB, and boot into the full PlayOS experience from internal storage.

**Status:** 🟢 Implemented and verified — installer source (`src/playos-installer/`), `playos.mode=install` trigger, complete `FactoryReset` handling, Buildroot installer-image wiring, and T8 validation (QEMU loopback + physical Ally install) have all landed; user confirmed NVMe install + boot from internal storage.

**Prerequisites:** Sprint 9 complete — complete MVP feature set running on the ROG Ally.

---

## Why This Sprint Exists

Sprint 9 completes the feature set. Sprint 10 makes PlayOS installable to real hardware — without it, the system only runs from USB. This sprint also completes the `FactoryReset` IPC (all options) and establishes the full 5-partition installed-disk layout (ESP, A/B system slots, `misc`, data) that Sprint 11 builds the A/B update and rollback logic on.

---

## Start Condition Checklist

- Sprint 9 complete: full feature set running on the ROG Ally from USB.
- `FactoryReset { erase_cache, erase_config }` is implemented (Sprint 6); `erase_games`, `erase_saves`, and `erase_logs` are deferred to this sprint.
- `playos-refdistro` produces a bootable USB image (`make ally-usb-image` → `output/ally/images/playos-ally-usb.img`).
- QEMU can boot the dev image on the host for testing (`make qemu-run`).
- A spare NVMe or a test-only Ally is available for destructive disk tests.

---

## Decisions Locked for This Sprint

- **Partition layout (installed disk, 5 partitions):** ESP (512 MiB FAT32, label `ESP`), system A (4 GiB EROFS/squashfs, read-only, label `playos-a`), system B (4 GiB EROFS/squashfs, read-only, label `playos-b`, empty until Sprint 11), `misc` (64 MiB, A/B slot metadata, label `misc`), data (remainder ext4, label `playos-data`). The live USB keeps its current compact 3-partition layout — this 5-partition layout applies only to the installer-deployed internal disk.
- **Installer trigger:** `playos.mode=install` kernel cmdline flag, OR no existing PlayOS data partition on a removable-boot device
- **Installer NEVER silently formats:** user must see disk details and hold A for 3 seconds
- **Disk partitioning tool:** `libfdisk` or raw ioctl (no parted/fdisk subprocess)
- **UEFI fallback:** always write `/EFI/BOOT/BOOTX64.EFI`; `efibootmgr` registration is a best-effort addition
- **Factory reset authority:** only `playos-init` performs the filesystem deletion; the overlay sends an IPC command

---

## Scope

### In Scope

- Installer trigger mode in `playos-init`
- Installer Raylib UI (disk discovery, confirmation, progress, success, error screens)
- GPT partitioning via `libfdisk`
- FAT32 ESP format (`mkfs.fat`)
- Read-only system A slot from a pre-built EROFS/squashfs image (`playos-a`)
- Empty system B slot reserved for Sprint 11 A/B (`playos-b`)
- A/B metadata partition (`misc`, 64 MiB)
- ext4 data partition format (`mkfs.ext4`)
- EFI boot artifact write
- UEFI boot entry registration (`efibootmgr`, best-effort)
- First-boot from internal disk
- Complete `FactoryReset` IPC (all five options)
- `make installer-image` Buildroot target

### Explicitly Out of Scope

- A/B update/rollback mechanism and dm-verity hashing (Sprint 11) — Sprint 10 only creates the partitions
- Network update download (Sprint 11/post-MVP)
- dm-verity (Sprint 11/post-MVP)
- Windows dual-boot or partition preservation (explicitly out of MVP scope)

---

## Required Repository Changes

| Repo | Required work |
|---|---|
| `playos-refdistro` | `playos-installer` source, Buildroot package, installer image target, disk ops; overlay factory-reset UI flow |
| `playos-init` | Complete `FactoryReset` server handler (games/saves/logs erasure); installer trigger mode |
| `playos-runtime` | `playos_trusted_factory_reset()` client helper |
| `playos-spec` | Installation guide, partition layout doc |

---

## Expected Files and Directories

### `playos-refdistro`

```text
src/playos-installer/
    main.c                   # screen state machine
    screens/discovery.c
    screens/confirmation.c
    screens/progress.c
    screens/success.c
    screens/error.c
    disk/partition.c         # libfdisk wrapper
    disk/format.c            # mkfs.fat (ESP), mkfs.ext4 (data/misc); system slots written from pre-built EROFS/squashfs images
    disk/efi.c               # EFI artifact copy, efibootmgr
br2-external/package/playos-installer/
    playos-installer.mk
    Config.in
br2-external/configs/playos_ally_installer_defconfig
```

> **Note:** Like `src/playos-overlay/` (Sprint 7), `src/playos-installer/` is a second deliberate in-refdistro C-source exception. When implemented, update `playos-refdistro/AGENTS.md` "What NOT to Do" to list it alongside the overlay exception.

---

## Agent Task Breakdown

### Task Status Grid

| Task ID | Task | Primary repo | Status | Notes / evidence |
|---|---|---|---|---|
| S10-T1 | Implement installer trigger mode in `playos-init` | `playos-init` | done | `mount.c:421-455` parses `playos.mode=install`; `main.c`/`supervisor.c` spawn `/usr/bin/playos-installer` |
| S10-T2 | Build installer screen state machine (Raylib UI) | `playos-refdistro` | done | `src/playos-installer/main.c` DISK_DISCOVERY → CONFIRMATION → INSTALLING → SUCCESS/ERROR |
| S10-T3 | Implement disk partitioning and formatting | `playos-refdistro` | done | `disk.c` (libfdisk GPT), `format.c` (mkfs.fat/mkfs.ext4, squashfs slot write) |
| S10-T4 | Implement EFI artifact write and UEFI boot entry | `playos-refdistro` | done | `efi.c` writes `/EFI/BOOT/BOOTX64.EFI`, best-effort `efibootmgr` |
| S10-T5 | Implement first-boot from internal disk | `playos-init` | done | Existing S6 first-boot provisioning already handles empty `/data` (`mount.c` `.playos-storage-version` marker + seed games); no new code required |
| S10-T6 | Complete `FactoryReset` IPC (all five options) | `playos-init`, `playos-runtime`, `playos-refdistro` | done | Handler now erases `games`/`saves`/`cache`/`config`/`logs`; runtime trusted helper added |
| S10-T7 | Create `make installer-image` Buildroot target | `playos-refdistro` | done | `playos_ally_installer_defconfig`, `playos-installer` package, `linux-installer.config`, `scripts/gen-installer-usb-image.sh`, Makefile `installer-*` targets |
| S10-T8 | Installer validation (QEMU loopback + Ally) | `playos-refdistro` | done | QEMU loopback install SUCCESS (GPT verified); dev variant booted; physical Ally install to NVMe + reboot confirmed by user; **2026-08-19 re-install on an already-installed NVMe fixed** — `playos-init` now skips ESP mount and pivot in installer mode, and `playos-installer` appends step diagnostics to `/data/log/installer.log` |

### S10-T1 — Implement installer trigger mode in `playos-init`

- Parse kernel cmdline for `playos.mode=install`
- If absent: detect whether current root is a removable device and no `playos-data` partition exists on an internal disk; if both true, trigger installer mode
- In installer mode: spawn `playos-installer` as the first and only Wayland client (compositor must not spawn the shell)
- Log reason for entering installer mode

**Done when:** `playos.mode=install` on the cmdline boots into the installer UI, not the shell.

### S10-T2 — Build installer screen state machine (Raylib UI)

State machine: `DISK_DISCOVERY` → `CONFIRMATION` → `INSTALLING` → `SUCCESS` | `ERROR`

**Disk discovery screen:**
- Enumerate `/sys/block/` for NVMe and SATA devices
- Exclude the device currently hosting the root filesystem (the USB)
- Show for each candidate: model name, total size in GB, current partition count
- D-pad navigates; A selects

**Confirmation screen:**
- `WARNING: All data on <model> (<size> GB) will be erased`
- Hold A for 3 seconds to confirm (show countdown bar)
- B returns to disk discovery

**Progress screen:**
- Numbered step list with current step highlighted
- Progress bar (0–100%)
- Steps: Create GPT → Create ESP → Create system A → Create system B → Create misc → Create data → Write EFI → Populate system A → Format data → Init data → Sync

**Success screen:**
- "Installation complete. Remove USB and press A to restart."
- A: reboot

**Error screen:**
- Show failed step and error message
- Options: "Retry" (restart from disk discovery), "Shutdown"
- Never drop to a shell

**Done when:** all screens render, navigation works, and the full flow can be exercised in QEMU on a loopback device.

### S10-T3 — Implement disk partitioning and formatting

Using `libfdisk`:
1. Create GPT partition table on the target device
2. Partition 1: EFI System Partition, FAT32, 512 MiB, label `ESP`
3. Partition 2: system A, 4 GiB, label `playos-a` (read-only EROFS/squashfs root slot)
4. Partition 3: system B, 4 GiB, label `playos-b` (reserved empty for Sprint 11 A/B)
5. Partition 4: `misc`, 64 MiB, label `misc` (A/B slot metadata)
6. Partition 5: data, ext4, remainder, label `playos-data`
7. Write partition table to disk

Format/populate:
- ESP: call `mkfs.fat -F32 -n ESP <part1>`; copy `EFI/BOOT/BOOTX64.EFI`
- System A: write the pre-built read-only root image (EROFS, fallback squashfs) directly to `<part2>` — no `mkfs` at install time
- System B: leave empty/blank; populated by the Sprint 11 A/B update path
- `misc`: call `mkfs.ext4 -L misc <part4>` (or leave raw; the slot state is tiny)
- Data: call `mkfs.ext4 -L playos-data <part5>`

Each step must check the return code. On any failure: transition to ERROR screen with the error code and step name.

**Done when:** QEMU loopback test shows the correct 5-partition GPT layout, system A boots read-only, and `playos-data` mounts.

### S10-T4 — Implement EFI artifact write and UEFI boot entry

1. Mount the ESP at `/mnt/efi`
2. Create `/mnt/efi/EFI/BOOT/`
3. Copy the PlayOS EFI-stub kernel (bzImage with embedded initramfs) to `BOOTX64.EFI` — no intermediate bootloader (matches `scripts/gen-ally-usb-image.sh`)
4. Run `efibootmgr --create --disk <dev> --part 1 --label "PlayOS" --loader /EFI/BOOT/BOOTX64.EFI` — log success or failure; failure is non-fatal (fallback path works)
5. Unmount ESP

**Done when:** after install, removing the USB and rebooting loads PlayOS from the NVMe EFI artifact.

### S10-T5 — Implement first-boot from internal disk

On first boot from internal disk (empty `/data`):
- `playos-init` detects no `.playos-storage-version` marker
- Runs the full first-boot provisioning from Sprint 6 (S6-T1/T2)
- Boots into the shell with an empty (but valid) game library

**Done when:** after installation and reboot, the shell shows an empty library with the correct status bar.

### S10-T6 — Complete `FactoryReset` IPC

Extend the Sprint 6 handler (which already erases `cache` and `config`) to act on all five flags:

```json
{
  "v": 1,
  "type": "FactoryReset",
  "erase_games": false,
  "erase_saves": false,
  "erase_cache": true,
  "erase_config": true,
  "erase_logs": false
}
```

- `erase_games`  → delete contents of `/data/games/`
- `erase_saves`  → delete contents of `/data/saves/` (DESTRUCTIVE)
- `erase_cache`  → delete contents of `/data/cache/`
- `erase_config` → delete contents of `/data/config/`
- `erase_logs`   → delete contents of `/data/log/`

- Requires no active game; if a game is running, reply `FactoryResetError` with `"reason": "game_running"` (matches `runtime-ipc.md`).
- Success replies `FactoryResetComplete`; the erased directories are recreated empty.
- `erase_saves` is destructive: overlay must show a second confirmation before sending the IPC.
- Factory reset with `erase_games = true` and `erase_saves = true` leaves the system bootable with an empty game library and a valid `/data` tree.
- Access: Overlay → Power menu → "Factory Reset" → option selection → confirmation.

**Done when:** full factory reset (all true) leaves a bootable system with an empty shell library.

### S10-T7 — Create `make installer-image` Buildroot target

- `playos_ally_installer_defconfig` — like `playos_ally_defconfig` but with `playos-installer` instead of `playos-shell` as the first Wayland client
- `make installer-image` produces a bootable USB image that enters installer mode
- `make ally-usb-image` continues to produce the normal system image (unchanged)
- Document both targets in `README.md`

**Done when:** `make installer-image` completes without errors; the produced image boots into installer mode in QEMU.

### S10-T8 — Installer validation

QEMU loopback test:
- `make installer-image`; boot in QEMU with a loopback block device as target
- Verify: disk discovery shows the loopback device, confirmation screen works, partition layout correct after install, data filesystem mounts

Physical Ally test:
- Full install from USB to NVMe; remove USB; reboot; verify PlayOS starts from internal storage
- Install a game manually (`cp -r` to `/data/games/`); reboot; verify it appears in the shell

Abort test:
- Kill installer mid-progress; verify device is in a deterministic (recoverable) state

Re-install test:
- Run installer on an already-installed Ally; verify it works
- **2026-08-19 fix:** on a previously-installed NVMe, installer mode in `playos-init` now skips mounting the internal `ESP` label (the disk is about to be repartitioned) and skips `playos_pivot_to_active_slot()` (so `/usr/bin/playos-installer` stays visible). Without this, re-install failed with `Installation Failed` because the old ESP mount made the later `fdisk`/`mkfs` steps fail busy. Verified end-to-end by user: re-install completed and booted from NVMe.

**Done when:** all four validation scenarios pass with evidence.

---

## Implementation Guidance

### Disk operations ordering

Always sync after each major step: after partition table write, after each `mkfs`, after EFI copy. Use `fsync()` on file descriptors and `sync()` system call before unmounting.

### Never fork without exec

Use `posix_spawn` or `execve` for `mkfs.fat`, `mkfs.ext4`, `efibootmgr`. Do not use `system()`.

### libfdisk over shell

The installer runs as root inside the initramfs. Use `libfdisk` directly — do not depend on `fdisk` or `parted` binaries being present.

---

## Verification and Evidence

| Evidence | How it is produced |
|---|---|
| Partition layout | `fdisk -l <device>` after QEMU loopback install |
| Filesystem types | `blkid` output showing FAT32 (ESP), EROFS/squashfs (system slots), ext4 (data/misc) labels |
| NVMe boot | ROG Ally boot without USB, shell visible |
| Factory reset | `ls /data/games/` empty after full reset; system boots |
| Installer error screen | Write-protect target in QEMU; verify error screen appears |

---

## Acceptance Criteria

- [x] Installer boots from USB and shows disk discovery screen
- [x] Only internal NVMe disks listed; USB boot device excluded
- [x] Disk model, size, and partition count shown correctly
- [x] Confirmation requires holding A for 3 seconds (no accidental erase)
- [x] B on confirmation returns to disk selection
- [x] Installation completes on clean NVMe without error
- [x] Progress bar reaches 100% and success screen appears
- [x] After USB removal and reboot: PlayOS boots from NVMe
- [x] Empty game library shown on first boot from internal disk
- [x] File written to `/data/` persists after restart
- [x] Manually installed game appears in shell after restart
- [x] Full factory reset leaves system bootable with empty valid data partition
- [x] Error screen shown if installation fails at any step (no terminal drop)
- [x] `make installer-image` produces a bootable installer USB image
- [x] CI passes (installer compiles; disk operations tested on QEMU loopback)

---

## Handoff to Sprint 11

Sprint 11 may assume:

- The installer creates the full 5-partition layout (ESP, system A/B, `misc`, data)
- System A is populated with the read-only EROFS/squashfs root and boots; system B is reserved empty; `misc` exists
- `playos-init` boots from the ESP EFI artifact (BOOTX64.EFI) and provisions `/data`
- `/data` provisioning is stable and tested
- Sprint 11 implements A/B update/rollback logic and dm-verity on top of the existing layout; the installer does not repartition

---

## Exit Gate

A PlayOS USB installer image successfully installs PlayOS to the ROG Ally internal NVMe. After removal of the USB drive, the Ally boots PlayOS from internal storage and shows the shell.

*Previous: [Sprint 9](Sprint-9.md) | Next: [Sprint 11](Sprint-11.md)*
