# PlayOS Installation Guide

> **Version:** 1.0
> **Applies to:** Sprint 10 installer path (ROG Ally x86_64 target).

This guide covers building the one-shot installer USB image and using it to install
PlayOS onto the ROG Ally internal NVMe SSD. The normal live-USB image (boot directly
from USB without installing) is documented in the
[Build Guide](build-guide.md).

---

## 1. Overview

The installer is a separate, one-shot system image:

- The **installer kernel** is the normal Ally kernel with
  `CONFIG_CMDLINE="... playos.mode=install"`.
- On boot, `playos-init` sees `playos.mode=install` and spawns
  `/usr/bin/playos-installer` instead of `playos-shell`.
- The installer shows a Raylib confirmation UI, then writes the five-partition
  internal layout to the target NVMe and stages the production kernel and system
  image.

The target medium is a **fixed disk** (removable flag `0`), so the boot USB is
automatically excluded from the disk-selection list.

---

## 2. Build the installer image

```bash
cd playos-refdistro
make installer-image
```

This builds both the installer output (`output/installer/`) and the production Ally
output (`output/ally/`), then assembles:

```text
output/installer/images/playos-ally-installer.img
```

The production build must also produce a squashfs system image
(`BR2_TARGET_ROOTFS_SQUASHFS=y`), which becomes the installer payload
`playos-a/rootfs.squashfs`.

---

## 3. Flash the installer USB

```bash
make installer-flash
# or manually:
sudo dd if=output/installer/images/playos-ally-installer.img \
        of=/dev/sdX bs=4M status=progress conv=fsync
```

> **Warning:** `of=/dev/sdX` must be the USB device (whole disk), not a partition.
> All data on the USB is overwritten.

---

## 4. Install to the Ally

1. Power off the Ally.
2. Insert the installer USB.
3. Boot the Ally and select the USB as the UEFI boot device.
4. The installer discovers internal fixed disks and shows model/size/partition count.
5. Use the D-pad to select the target NVMe and press **A**.
6. Hold **A** on the confirmation screen until the countdown bar completes.
7. Wait for installation to reach 100% and show the success screen.
8. Remove the USB and reboot.

The Ally then boots from the internal ESP `EFI/BOOT/BOOTX64.EFI`.

---

## 5. What the installer writes

| Step | Action |
|---:|---|
| 1 | Create GPT partition table on the target disk |
| 2 | Format partition 1 as FAT32 (`ESP`, 512 MiB) |
| 3 | Write the squashfs system image to `playos-a` (4 GiB) |
| 4 | Reserve `playos-b` (4 GiB) empty |
| 5 | Format `misc` (64 MiB, ext4) |
| 6 | Format `playos-data` (remainder, ext4) |
| 7 | Write `EFI/BOOT/BOOTX64.EFI` to the ESP |
| 8 | Sync all filesystems and best-effort `efibootmgr` NVRAM entry |

See [Partition Layout](partition-layout.md) for the full layout.

---

## 6. Boot model (Sprint 11.5)

The installed system boots via the EFI-stub kernel and its initramfs, which then mounts
the active slot's read-only squashfs image (`playos-a` by default) and pivots into it
(Sprint 11.5). In practice this means:

- Removing the USB and booting from the internal ESP works now.
- `/data` first-boot provisioning runs from the initramfs as usual.
- The active-slot squashfs is the booted read-only root; the inactive slot (`playos-b`)
  is used for A/B updates and rollback.

---

## 7. Future work

- dm-verity integrity hardening (Sprint 12+).
- Automatic installer triggering when a removable boot medium has no `playos-data`
  on an internal disk (currently the trigger is the explicit `playos.mode=install`
  command line only).

---

*Related: [Partition Layout](partition-layout.md), [Build Guide](build-guide.md),
[Sprint 10](sprints/Sprint-10.md)*
