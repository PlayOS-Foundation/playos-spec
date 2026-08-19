# PlayOS Kernel Configuration Guide

> **Repository:** `playos-refdistro/br2-external/configs/`  
> **Cross-references:** [build-guide.md](build-guide.md), [Sprint-3.md](Sprint-3.md), [architecture.md](architecture.md) §18

---

## Philosophy

**Start from a working ROG Ally configuration and remove features gradually.**  
Do not begin from an aggressively minimal embedded configuration — you will spend weeks re-enabling drivers you removed.

Only remove a subsystem after confirming it is absent from the supported hardware list and no test regression occurs.

---

## Kernel Branch Policy

| Track | Purpose |
|---|---|
| `playos-kernel-lts` | Release and qualification — upstream Linux LTS |
| `playos-kernel-next` | Hardware evaluation — newer stable branch |

The LTS branch is the shipping kernel. The next branch is used to evaluate new hardware enablement (new Ally revisions, Intel support) before promotion to LTS.

---

## Required Subsystems

### Core x86_64 and Boot

```kconfig
CONFIG_X86_64=y
CONFIG_EFI=y
CONFIG_EFI_STUB=y           # Kernel acts as its own EFI loader
CONFIG_BLK_DEV_INITRD=y     # Embedded initramfs
CONFIG_INITRAMFS_SOURCE=""  # Buildroot fills this in
CONFIG_ACPI=y
CONFIG_ACPI_SLEEP=y
CONFIG_PCI=y
CONFIG_PCI_MSI=y
CONFIG_PCIEPORTBUS=y
CONFIG_AMD_IOMMU=y          # ROG Ally uses AMD IOMMU
CONFIG_INTEL_IOMMU=y        # Intel target
```

### Virtual Filesystems (required for PID 1)

```kconfig
CONFIG_DEVTMPFS=y
CONFIG_DEVTMPFS_MOUNT=y
CONFIG_PROC_FS=y
CONFIG_SYSFS=y
CONFIG_TMPFS=y
CONFIG_TMPFS_POSIX_ACL=y
CONFIG_CGROUPS=y            # optional but useful for future process isolation
```

### Storage

```kconfig
CONFIG_BLK_DEV_NVME=y       # ROG Ally internal SSD
CONFIG_EFI_PARTITION=y
CONFIG_MSDOS_PARTITION=y    # for USB drives
CONFIG_VFAT_FS=y            # ESP (FAT32)
CONFIG_EXT4_FS=y            # Data partition
CONFIG_EXT4_USE_FOR_EXT2=y
CONFIG_SQUASHFS=y           # Read-only system root (A/B squashfs slots)
# CONFIG_XFS_FS is not needed
# CONFIG_BTRFS_FS is not needed
```

### Graphics — AMD (ROG Ally)

```kconfig
CONFIG_DRM=y
CONFIG_DRM_KMS_HELPER=y
CONFIG_DRM_AMDGPU=y
CONFIG_DRM_AMD_DC=y         # AMD Display Core — required for DisplayPort/eDP
CONFIG_DRM_AMD_DC_DCN=y     # DCN (Display Core Next) for RDNA GPUs
CONFIG_DRM_SIMPLEDRM=y      # Firmware framebuffer fallback (recovery mode)
CONFIG_FRAMEBUFFER_CONSOLE=n # No VT framebuffer console in production
CONFIG_DRM_FBDEV_EMULATION=n
```

### Graphics — Intel (Sprint 13)

```kconfig
# For Gen 9–12 (Ice Lake, Tiger Lake, Alder Lake):
CONFIG_DRM_I915=y
# For Gen 12.5+ (Meteor Lake, Lunar Lake) — choose one:
# CONFIG_DRM_XE=y
```

### USB and Input

```kconfig
CONFIG_USB=y
CONFIG_USB_XHCI_HCD=y       # USB 3.x host controller
CONFIG_USB_HID=y
CONFIG_HID=y
CONFIG_HID_GENERIC=y
CONFIG_HID_ASUS=y            # ROG Ally vendor-specific HID quirks
CONFIG_INPUT=y
CONFIG_INPUT_EVDEV=y         # evdev interface for playos-platform-api
CONFIG_INPUT_JOYSTICK=y
CONFIG_JOYSTICK_XPAD=n       # We use HID not xpad for the Ally
```

### Audio — ROG Ally (AMD ACP / HDA)

```kconfig
CONFIG_SOUND=y
CONFIG_SND=y
CONFIG_SND_HDA_INTEL=y
CONFIG_SND_HDA_CODEC_REALTEK=y   # ROG Ally uses Realtek codec
CONFIG_SND_SOC=y
CONFIG_SND_SOC_AMD_ACP=y         # AMD Audio Co-Processor
CONFIG_SND_SOC_AMD_MACH=y        # or specific machine driver
# CONFIG_SND_USB_AUDIO=y         # Add for USB audio adapter support
```

### Networking (development / Wi-Fi post-MVP)

```kconfig
# Minimal for first sprints
CONFIG_NET=y
CONFIG_UNIX=y                    # Required for Wayland sockets

# Sprint 11.6 — wired developer SSH (USB-C Ethernet + QEMU test NIC).
# No wireless in this sprint:
CONFIG_NETDEVICES=y
CONFIG_ETHERNET=y
CONFIG_MII=y
CONFIG_USB_NET_DRIVERS=y
CONFIG_USB_USBNET=y
CONFIG_USB_NET_AX8817X=y
CONFIG_USB_NET_AX88179_178A=y
CONFIG_USB_RTL8152=y
CONFIG_USB_NET_CDCETHER=y
CONFIG_USB_NET_CDC_NCM=y
CONFIG_USB_NET_CDC_EEM=y
CONFIG_USB_NET_RNDIS_HOST=y
CONFIG_USB_NET_SMSC95XX=y
CONFIG_USB_NET_MCS7830=y
CONFIG_VIRTIO_NET=y
CONFIG_E1000=y
CONFIG_E1000E=y

# Wi-Fi deferred to post-MVP / Sprint 16:
# CONFIG_CFG80211=y
# CONFIG_MAC80211=y
# CONFIG_IWLWIFI=y               # Intel Wi-Fi
# CONFIG_ATH11K=y / CONFIG_MT7921E=y  # AMD/Mediatek (Ally has AMD Wi-Fi)
```

### Power Management

```kconfig
CONFIG_ACPI_BATTERY=y
CONFIG_POWER_SUPPLY=y
CONFIG_THERMAL=y
CONFIG_THERMAL_HWMON=y
CONFIG_X86_THERMAL_VECTOR=y
CONFIG_X86_AMD_PSTATE=y          # AMD P-state driver (ROG Ally)
CONFIG_X86_AMD_PSTATE_UT=n       # Skip unit test driver
CONFIG_X86_INTEL_PSTATE=y        # Intel P-state (Sprint 13)
CONFIG_CPU_FREQ=y
CONFIG_CPU_FREQ_GOV_PERFORMANCE=y
CONFIG_CPU_FREQ_GOV_POWERSAVE=y
CONFIG_SUSPEND=y                  # System suspend (post-MVP full support)
CONFIG_PM_SLEEP=y
CONFIG_HIBERNATION=n              # Not needed for MVP
```

### Watchdog

```kconfig
CONFIG_WATCHDOG=y
CONFIG_WATCHDOG_NOWAYOUT=y
CONFIG_SP5100_TCO=y              # AMD SB800 / FCH watchdog (ROG Ally)
```

### Security (Sprint 12)

```kconfig
CONFIG_SECCOMP=y
CONFIG_SECCOMP_FILTER=y
CONFIG_SECURITY=y
CONFIG_SECURITY_LANDLOCK=y       # Landlock LSM
CONFIG_LSM="landlock,lockdown,yama"
CONFIG_DM_VERITY=y               # dm-verity for system image integrity
CONFIG_DM_VERITY_VERIFY_ROOTHASH_SIG=n  # Signature check optional initially
CONFIG_MODULE_SIG=y              # Module signing
CONFIG_MODULE_SIG_ALL=y
```

### EFI Variables (needed for A/B boot counting)

```kconfig
CONFIG_EFI_VARS=y
CONFIG_EFIVAR_FS=y
```

---

## What to Remove

Only after verifying the feature is not used by any supported hardware:

```kconfig
# Server filesystems
CONFIG_XFS_FS=n
CONFIG_BTRFS_FS=n
CONFIG_NFS_FS=n
CONFIG_NFSD=n

# Unused network protocols
CONFIG_IPV6=n       # unless needed for Wi-Fi
CONFIG_NETFILTER=n  # no iptables

# Unused GPU drivers
CONFIG_DRM_RADEON=n     # Legacy AMD — not RDNA
CONFIG_DRM_NOUVEAU=n    # NVIDIA
CONFIG_DRM_VMWGFX=n     # VMware

# Legacy buses
CONFIG_ISA=n
CONFIG_PARPORT=n
CONFIG_PCMCIA=n

# Virtualization host features
CONFIG_KVM=n
CONFIG_XEN=n

# Debug features (production builds)
CONFIG_DEBUG_KERNEL=n
CONFIG_KGDB=n
CONFIG_SLUB_DEBUG=n
CONFIG_KALLSYMS=n       # Remove after bring-up is stable
```

---

## Config File Locations

```
br2-external/configs/playos_qemu_x86_64_defconfig    QEMU target
br2-external/configs/playos_rog_ally_defconfig        ROG Ally (primary)
br2-external/configs/playos_intel_pc_defconfig        Intel (Sprint 13)
```

Each defconfig sets `BR2_LINUX_KERNEL_CUSTOM_CONFIG_FILE` to point to a kernel config fragment in `board/playos/<target>/linux.config`.

---

## Kernel Config Workflow

```bash
# Edit kernel config interactively
make -C buildroot O=build/rog-ally linux-menuconfig

# Save changes back to the fragment
make -C buildroot O=build/rog-ally linux-update-defconfig

# Check for missing dependencies
make -C buildroot O=build/rog-ally linux-check-package
```

---

## Firmware

ROG Ally requires AMD GPU firmware. Firmware blobs are non-redistributable and must be sourced from:
- An existing ROG Ally Linux installation: `/lib/firmware/amdgpu/`
- The linux-firmware repository (check licensing)

In Buildroot, firmware is placed in:
```
board/playos/rog-ally/rootfs-overlay/lib/firmware/amdgpu/
```

Required blobs (exact filenames vary by GPU revision — check `dmesg` for requests):
```
amdgpu/gc_11_0_4_pfp.bin
amdgpu/gc_11_0_4_me.bin
amdgpu/gc_11_0_4_ce.bin
amdgpu/gc_11_0_4_rlc.bin
amdgpu/gc_11_0_4_mec.bin
amdgpu/dcn_3_1_4_dmcub.bin
... (and others — check dmesg on first boot)
```

CPU microcode:
```
board/playos/rog-ally/rootfs-overlay/lib/firmware/amd-ucode/microcode_amd_fam19h.bin
```
