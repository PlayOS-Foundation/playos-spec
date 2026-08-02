# Sprint 10 — Installer and Internal-Disk Deployment

**Goal:** Build a tested, user-confirmed installation path from a USB boot image to the ROG Ally internal NVMe SSD. After installation, the Ally boots PlayOS from the internal disk without a USB drive.

**Primary Outcome:** A user can boot the PlayOS installer from USB, confirm the target disk, wait for installation, remove the USB, and boot into the full PlayOS experience from internal storage.

**Prerequisites:** Sprint 9 complete — complete MVP feature set running on the ROG Ally.

---

## Key Deliverables

### Installer Design

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
