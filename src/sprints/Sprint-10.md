# Sprint 10 — Installer and Internal-Disk Deployment

**Goal:** Build a tested, user-confirmed installation path from a USB boot image to the ROG Ally internal NVMe SSD. After installation, the Ally boots PlayOS from the internal disk without a USB drive.

**Primary Outcome:** A user can boot the PlayOS installer from USB, confirm the target disk, wait for installation, remove the USB, and boot into the full PlayOS experience from internal storage.

**Prerequisites:** Sprint 9 complete — complete MVP feature set running on the ROG Ally.

---

## Why This Sprint Exists

Sprint 9 completes the feature set. Sprint 10 makes PlayOS installable to real hardware — without it, the system only runs from USB. This sprint also completes the `FactoryReset` IPC (all options) and establishes the partition layout that Sprint 11 will extend to A/B.

---

## Start Condition Checklist

- Sprint 9 complete: full feature set running on the ROG Ally from USB.
- `FactoryReset { erase_cache, erase_config }` is implemented (Sprint 6); `erase_games` and `erase_saves` are stubs.
- `playos-refdistro` produces a bootable USB image (`make playos-image`).
- QEMU can boot the USB image on the host for testing.
- A spare NVMe or a test-only Ally is available for destructive disk tests.

---

## Decisions Locked for This Sprint

- **Partition layout (Sprint 10 version):** 3 partitions — EFI (512 MB FAT32), system (2 GB ext4), data (remainder ext4); Sprint 11 will expand to A/B
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
- ext4 data partition format (`mkfs.ext4`)
- EFI boot artifact write
- UEFI boot entry registration (`efibootmgr`, best-effort)
- First-boot from internal disk
- Complete `FactoryReset` IPC (all five options)
- `make installer-image` Buildroot target

### Explicitly Out of Scope

- A/B partition layout (Sprint 11)
- Network update download (Sprint 11/post-MVP)
- dm-verity (Sprint 11/post-MVP)
- Windows dual-boot or partition preservation (explicitly out of MVP scope)

---

## Required Repository Changes

| Repo | Required work |
|---|---|
| `playos-refdistro` | Installer source, Buildroot package, installer image target, disk ops |
| `playos-runtime` | Complete `FactoryReset` IPC, install-mode signaling |
| `playos-overlay` | Factory reset UI flow in power menu |
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
    disk/format.c            # mkfs.fat, mkfs.ext4 subprocess calls
    disk/efi.c               # EFI artifact copy, efibootmgr
br2-external/package/playos-installer/
    playos-installer.mk
    Config.in
br2-external/configs/playos_ally_installer_defconfig
```

---

## Agent Task Breakdown

### Task Status Grid

| Task ID | Task | Primary repo | Status | Notes / evidence |
|---|---|---|---|---|
| S10-T1 | Implement installer trigger mode in `playos-init` | `playos-refdistro` | not started | |
| S10-T2 | Build installer screen state machine (Raylib UI) | `playos-refdistro` | not started | |
| S10-T3 | Implement disk partitioning and formatting | `playos-refdistro` | not started | |
| S10-T4 | Implement EFI artifact write and UEFI boot entry | `playos-refdistro` | not started | |
| S10-T5 | Implement first-boot from internal disk | `playos-refdistro` | not started | |
| S10-T6 | Complete `FactoryReset` IPC (all five options) | `playos-runtime`, `playos-overlay` | not started | |
| S10-T7 | Create `make installer-image` Buildroot target | `playos-refdistro` | not started | |
| S10-T8 | Installer validation (QEMU loopback + Ally) | `playos-refdistro` | not started | |

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
- Steps: Create GPT → Create EFI → Create system → Create data → Write EFI → Format data → Init data → Sync

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
2. Partition 1: EFI System Partition, FAT32, 512 MB, label `EFI`
3. Partition 2: PlayOS system, ext4, 2 GB, label `playos-system`
4. Partition 3: PlayOS data, ext4, remainder, label `playos-data`
5. Write partition table to disk

Format:
- ESP: call `mkfs.fat -F32 -n EFI <part1>`
- System: call `mkfs.ext4 -L playos-system <part2>` (blank; will be written in S10-T4)
- Data: call `mkfs.ext4 -L playos-data <part3>`

Each step must check the return code. On any failure: transition to ERROR screen with the error code and step name.

**Done when:** QEMU loopback test shows correct 3-partition GPT layout and all three filesystems mountable.

### S10-T4 — Implement EFI artifact write and UEFI boot entry

1. Mount the ESP at `/mnt/efi`
2. Create `/mnt/efi/EFI/BOOT/`
3. Copy the PlayOS EFI artifact (kernel + initramfs combined UKI, or GRUB stub) to `BOOTX64.EFI`
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

Extend the Sprint 6 stub to support all five options:

```
FactoryReset {
  erase_games:  bool,   // delete /data/games/ contents
  erase_saves:  bool,   // DESTRUCTIVE — requires separate confirmation
  erase_cache:  bool,
  erase_config: bool,
  erase_logs:   bool
}
```

- `erase_saves` is destructive: overlay must show a second confirmation before sending the IPC
- Factory reset with `erase_games = true` and `erase_saves = true` leaves the system bootable with an empty, valid data partition
- Access: Overlay → Power menu → "Factory Reset" → option selection → confirmation

**Done when:** full factory reset (all true) leaves a bootable system with an empty shell library.

### S10-T7 — Create `make installer-image` Buildroot target

- `playos_ally_installer_defconfig` — like `playos_ally_defconfig` but with `playos-installer` instead of `playos-shell` as the first Wayland client
- `make installer-image` produces a bootable USB image that enters installer mode
- `make playos-image` continues to produce the normal system image (unchanged)
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
| Filesystem types | `blkid` output showing FAT32, ext4 labels |
| NVMe boot | ROG Ally boot without USB, shell visible |
| Factory reset | `ls /data/games/` empty after full reset; system boots |
| Installer error screen | Write-protect target in QEMU; verify error screen appears |

---

## Acceptance Criteria

- [ ] Installer boots from USB and shows disk discovery screen
- [ ] Only internal NVMe disks listed; USB boot device excluded
- [ ] Disk model, size, and partition count shown correctly
- [ ] Confirmation requires holding A for 3 seconds (no accidental erase)
- [ ] B on confirmation returns to disk selection
- [ ] Installation completes on clean NVMe without error
- [ ] Progress bar reaches 100% and success screen appears
- [ ] After USB removal and reboot: PlayOS boots from NVMe
- [ ] Empty game library shown on first boot from internal disk
- [ ] File written to `/data/` persists after restart
- [ ] Manually installed game appears in shell after restart
- [ ] Full factory reset leaves system bootable with empty valid data partition
- [ ] Error screen shown if installation fails at any step (no terminal drop)
- [ ] `make installer-image` produces a bootable installer USB image
- [ ] CI passes (installer compiles; disk operations tested on QEMU loopback)

---

## Handoff to Sprint 11

Sprint 11 may assume:

- The installer creates a working 3-partition layout
- `playos-init` reads the system partition and boots from it
- `/data` provisioning is stable and tested
- The EFI boot path is established (BOOTX64.EFI)
- Sprint 11 will expand the partition layout to A/B; the installer will be updated accordingly

---

## Exit Gate

A PlayOS USB installer image successfully installs PlayOS to the ROG Ally internal NVMe. After removal of the USB drive, the Ally boots PlayOS from internal storage and shows the shell.

*Previous: [Sprint 9](Sprint-9.md) | Next: [Sprint 11](Sprint-11.md)*

The installer runs as a special `playos-init` mode, triggered when:
- A kernel command line flag `playos.mode=install` is present, OR
- No existing PlayOS data partition is found and the device is running from a USB/removable medium

**Installer must never silently format.** The user must:
1. See the target disk, its model name, current partitions, and total size
2. Be warned that all data on the target disk will be erased
3. Explicitly confirm using the controller (hold A for 3 seconds, or confirm through a multi-step dialog)

### Installer Flow

```
Boot installer image from USB
    │
    ▼
playos-init detects install mode
    │
    ▼
Launch playos-installer (trusted Raylib client or direct Raylib app)
    │
    ▼
Disk Discovery Screen
    - Enumerate NVMe and SATA disks (via /sys/block/)
    - Show: model, size, current partition table
    - Exclude the boot USB device
    - User selects target disk (D-pad + A)
    │
    ▼
Confirmation Screen
    - "WARNING: All data on <disk model> (<size>) will be erased"
    - Controller: hold A for 3 seconds to confirm
    - B to cancel and return to disk selection
    │
    ▼
Installation Progress Screen
    1. Create GPT partition table
    2. Create EFI System Partition (FAT32, ~512MB)
    3. Create PlayOS system A partition (ext2/squashfs, ~2GB) [placeholder for A/B in Sprint 11]
    4. Create PlayOS data partition (ext4, remainder)
    5. Write /EFI/BOOT/BOOTX64.EFI to ESP
    6. Format data partition, create directory tree
    7. Write storage-version marker
    8. Sync filesystems
    - Progress bar (0–100%)
    │
    ▼
Success Screen
    - "Installation complete. Remove USB and press A to restart."
    - A button → system reboots
    │
    ▼ (USB removed)
ROG Ally boots from internal NVMe → PlayOS
```

### `playos-installer` — Raylib Installer UI

Create as a trusted Raylib client in `playos-refdistro/src/installer/`.

**Screens:** Disk discovery → confirmation → progress → success/failure

**Disk operations (run as root via `playos-init`):**
- Use `libfdisk` or raw `ioctl` for partition table manipulation
- Use `mkfs.fat` for ESP, `mkfs.ext4` for data partition
- Use `cp` or `dd` to write the boot artifact to ESP
- Verify each step; abort with a detailed error on any failure

**Error screen:**
- Show the failed step and error message
- Options: "Retry" (start over) or "Shutdown"
- Never drop to a shell

### Partition Layout

For this sprint, use a simple 3-partition layout (A/B expanded in Sprint 11):

```
GPT disk (ROG Ally NVMe)
├── Part 1: EFI System Partition   FAT32      512 MB    label: EFI
├── Part 2: PlayOS system          ext4/raw   2 GB      label: playos-system  (single slot for now)
└── Part 3: PlayOS data            ext4       remainder  label: playos-data
```

Note: Sprint 11 will expand `playos-system` to A/B slots.

### UEFI Boot Entry

After writing the ESP, optionally register an UEFI boot entry via `efibootmgr`:
```
efibootmgr --create --disk <nvme_dev> --part 1 --label "PlayOS" --loader /EFI/BOOT/BOOTX64.EFI
```

The fallback path `/EFI/BOOT/BOOTX64.EFI` works without a registered entry on most UEFI firmware.

### First-Boot After Installation

When PlayOS boots from the internal disk for the first time:
- `playos-init` finds no data content (empty `/data`)
- Creates the full directory tree
- Writes `.playos-storage-version`
- Boots normally into the shell (no games installed yet — empty library)

### Factory Reset (Complete Implementation)

Complete the `FactoryReset` IPC command from Sprint 6:

```
FactoryReset {
  erase_games: bool,      // deletes /data/games/ contents
  erase_saves: bool,      // deletes /data/saves/ contents  (DESTRUCTIVE — separate confirmation)
  erase_cache: bool,      // deletes /data/cache/ contents
  erase_config: bool,     // deletes /data/config/ contents
  erase_logs: bool        // deletes /data/logs/ contents
}
```

Access via: Overlay → Power menu → "Factory Reset" → confirmation screen

---

## Acceptance Criteria

- [ ] Installer boots from USB and shows the disk discovery screen
- [ ] Only internal NVMe disks are listed; USB boot device is excluded
- [ ] Disk model, size, and partition table are shown correctly
- [ ] Confirmation requires holding A for 3 seconds (no accidental erase)
- [ ] B on confirmation screen returns to disk selection
- [ ] Installation completes without error on a clean NVMe
- [ ] Progress bar reaches 100% and success screen appears
- [ ] After USB removal and reboot: PlayOS boots from NVMe
- [ ] Empty game library is shown on first boot from internal disk
- [ ] A file written to `/data/` on the internal disk persists after restart
- [ ] Installing a game manually (copying to `/data/games/`) and restarting shows it in the shell
- [ ] Factory reset (all options) leaves the system bootable with an empty but valid data partition
- [ ] Error screen is shown if installation fails at any step (simulate by write-protecting target)
- [ ] CI passes (installer compiles; disk operations tested on a loopback device in QEMU)

---

## Repositories Primarily Involved

| Repo | Work |
|---|---|
| `playos-refdistro` | `playos-installer` source, Buildroot package, installer image target (`make installer-image`) |
| `playos-runtime` | `FactoryReset` IPC full implementation, install-mode signaling |
| `playos-overlay` | Factory reset UI flow |
| `playos-spec` | Installation guide, partition layout documentation |

---

## Testing Approach

- Physical ROG Ally: full install flow from USB to NVMe
- Loopback test in QEMU: installer writes to a virtual disk; verify partition layout and filesystem
- Abort test: kill installer mid-progress; verify device is in a known (recoverable) state
- Re-install test: run installer on an already-installed device; verify it works correctly

---

## Exit Gate

A PlayOS USB installer image successfully installs PlayOS to the ROG Ally internal NVMe. After removal of the USB drive, the Ally boots PlayOS from internal storage and shows the shell.

*Previous: [Sprint 9](Sprint-9.md) | Next: [Sprint 11](Sprint-11.md)*
