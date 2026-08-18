# PlayOS Partition Layout

> **Version:** 1.0
> **Applies to:** Sprint 10+ installer-deployed internal disks and Sprint 6+ live USB media.

This document is the authoritative reference for on-disk partitioning. The layout is
different for the removable live-USB image and the internal NVMe install target; the
installer (`playos-installer`) creates the internal layout, while
`scripts/gen-ally-usb-image.sh` and `scripts/gen-installer-usb-image.sh` create the USB
layouts.

---

## 1. Internal disk (installed system)

The installer writes a GPT table with five partitions to the target fixed disk
(the non-USB NVMe/SATA drive):

| # | Label | Size | Filesystem | Purpose |
|---:|---|---|---|---|
| 1 | `ESP` | 512 MiB | FAT32 | EFI System Partition; `EFI/BOOT/BOOTX64.EFI` |
| 2 | `playos-a` | 4 GiB | squashfs (read-only) | Active system slot |
| 3 | `playos-b` | 4 GiB | squashfs (read-only) | Inactive system slot (reserved, Sprint 11) |
| 4 | `misc` | 64 MiB | ext4 | A/B slot metadata (`/data/misc`-style state) |
| 5 | `playos-data` | remainder | ext4 | Writable user data |

```text
GPT disk (internal NVMe/SATA)
├── Partition 1: ESP           512 MiB  FAT32     label "ESP"
├── Partition 2: playos-a      4 GiB    squashfs  label "playos-a"   (active slot)
├── Partition 3: playos-b      4 GiB    squashfs  label "playos-b"   (reserved)
├── Partition 4: misc          64 MiB   ext4      label "misc"       (A/B metadata)
└── Partition 5: playos-data   remainder ext4     label "playos-data" (writable)
```

**Rules:**

- `playos-a` and `playos-b` are immutable system slots. `misc` carries the tiny A/B
  slot state (which slot is booted / healthy); it is formatted ext4 for convenience.
- `playos-data` holds all user content: games, saves, cache, log, updates, config.
- The installer never silently formats a disk; the user must explicitly confirm the
  target with a hold-to-confirm gesture.
- Sprint 10 creates the slots; A/B update/rollback and dm-verity land in Sprint 11.

---

## 2. Live USB (normal system image)

`make ally-usb-image` produces `output/ally/images/playos-ally-usb.img` with a compact
three-partition GPT layout:

| # | Label | Size | Filesystem | Purpose |
|---:|---|---|---|---|
| 1 | `ESP` | 256 MiB | FAT32 | EFI System Partition; `EFI/BOOT/BOOTX64.EFI` |
| 2 | `playos-a` | 2048 MiB | ext2 | System image (EFI stub kernel lives on ESP) |
| 3 | `playos-data` | remainder | ext4 | Writable scratch / diagnostics |

This layout is intentionally simpler than the internal layout: the live image boots
entirely from its EFI-stub kernel with an embedded initramfs, and `playos-a` exists as
a labelled system container rather than as a booted root.

---

## 3. Installer USB

`make installer-image` produces `output/installer/images/playos-ally-installer.img`.
It reuses the three-partition live-USB geometry but carries a payload partition:

| # | Label | Size | Filesystem | Purpose |
|---:|---|---|---|---|
| 1 | `ESP` | 256 MiB | FAT32 | Installer kernel (`playos.mode=install`) as `EFI/BOOT/BOOTX64.EFI` |
| 2 | `playos-a` | 2048 MiB | ext2 | Install payload: `/rootfs.squashfs` + `/BOOTX64.EFI` |
| 3 | `playos-data` | remainder | ext4 | Scratch / preserved diagnostics |

The installer kernel boots the one-shot installer UI. It mounts its own `playos-a`
read-only and streams `rootfs.squashfs` into the target `playos-a` slot, and copies
`/BOOTX64.EFI` (the normal production kernel) into the target ESP.

---

## 4. Data partition contents

The writable `playos-data` partition is provisioned on first boot to:

```text
/data/
    games/<game-id>/          manifest.json, bin/, assets/, shaders/, licenses/
    saves/<game-id>/          profiles/, autosaves/, settings/
    cache/<game-id>/          shaders/, compiled-assets/, temporary/
    resources/
    downloads/
    log/
    updates/
    screenshots/
    config/
    profiles/
```

A `.playos-storage-version` marker is stamped at the root after first-boot
provisioning; its presence/absence drives first-boot seeding of shipped games.

---

*Related: [Installation Guide](installation-guide.md), [System Architecture](architecture.md#12-storage-layout)*
