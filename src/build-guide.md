# PlayOS Build Guide

> **Repository:** `playos-refdistro`  
> **Cross-references:** [dev-environment.md](dev-environment.md), [kernel-config.md](kernel-config.md), [Sprint-0.md](Sprint-0.md)

---

## Prerequisites

### Host system

Any recent Linux host (Ubuntu 22.04 LTS or later recommended). The Buildroot cross-compiler handles everything else.

```bash
# Ubuntu / Debian
sudo apt-get install -y \
  build-essential git wget curl unzip \
  libncurses-dev libssl-dev libelf-dev bc \
  python3 rsync cpio file \
  qemu-system-x86 ovmf \
  dosfstools mtools parted \
  cmake meson ninja-build \
  pkg-config wayland-scanner \
  libasound2-dev \
  sbsign pesign  # for EFI signing (Sprint 12+)
```

```bash
# Fedora / RHEL
sudo dnf install -y \
  @development-tools git wget \
  ncurses-devel openssl-devel elfutils-libelf-devel bc \
  python3 rsync cpio file \
  qemu edk2-ovmf \
  dosfstools mtools parted \
  cmake meson ninja-build pkg-config \
  wayland-devel
```

### `make setup`

The repository provides a `make setup` target that verifies host dependencies and installs any that are missing (using `apt-get` or `dnf`).

---

## Repository Layout

```
playos-refdistro/
├── buildroot/              official Buildroot (pinned git submodule)
├── br2-external/           PlayOS-specific Buildroot content
│   ├── external.desc
│   ├── Config.in
│   ├── external.mk
│   ├── configs/
│   │   ├── playos_qemu_x86_64_defconfig
│   │   ├── playos_rog_ally_defconfig
│   │   └── playos_intel_pc_defconfig
│   ├── board/playos/
│   │   ├── common/         post-build and post-image scripts shared across targets
│   │   ├── qemu-x86_64/    QEMU-specific overlays and scripts
│   │   └── rog-ally/       ROG Ally-specific overlays, firmware, scripts
│   ├── package/
│   │   ├── playos-init/
│   │   ├── playos-platform-api/
│   │   ├── playos-runtime/
│   │   ├── playos-compositor/
│   │   ├── playos-shell/
│   │   ├── playos-overlay/
│   │   └── raylib-playos/
│   └── patches/
│       ├── linux/          kernel patches (minimize; prefer upstream)
│       ├── wlroots/        wlroots patches if needed
│       └── raylib/         Raylib PlayOS backend patches
├── src/
│   └── playos-init/        PID 1 source (built as a Buildroot package)
├── protocols/              copy of playos-runtime Wayland protocol XML
├── scripts/                helper scripts
├── docs/ → ../             documentation (this file is one of them)
├── .github/workflows/      CI definitions
├── Makefile                developer command surface
└── versions.lock           pinned commits for all components
```

---

## Developer Commands

```bash
# Initial setup — installs host dependencies
make setup

# QEMU target
make qemu-config        # configure Buildroot for QEMU x86_64
make qemu-build         # full build (takes ~30–60 min on first run)
make qemu-run           # launch QEMU/OVMF with built image

# ROG Ally target
make ally-config        # configure Buildroot for ROG Ally
make ally-build         # full build
make ally-usb-image     # produce USB-bootable installer image

# Intel PC target
make intel-config
make intel-build
make intel-usb-image

# Installer image (for any target)
make installer-image TARGET=rog-ally

# Clean
make clean              # remove build outputs
make distclean          # remove everything including downloads cache
```

Behind each target:
```makefile
qemu-config:
	$(MAKE) -C buildroot O=$(BUILD_DIR)/qemu \
	    BR2_EXTERNAL=$(PWD)/br2-external \
	    playos_qemu_x86_64_defconfig

qemu-build:
	$(MAKE) -C buildroot O=$(BUILD_DIR)/qemu

qemu-run:
	scripts/run-qemu.sh $(BUILD_DIR)/qemu
```

Developers should not need to memorize raw Buildroot command lines.

---

## `versions.lock`

All external dependencies are pinned to full Git commit SHAs. **Never use floating branch names.**

```
# versions.lock
BUILDROOT_COMMIT=abc123...
LINUX_VERSION=6.6.30
LINUX_SOURCE=https://cdn.kernel.org/pub/linux/kernel/v6.x/linux-6.6.30.tar.xz
LINUX_SHA256=...

PLAYOS_PLATFORM_API_COMMIT=def456...
PLAYOS_RUNTIME_COMMIT=ghi789...
PLAYOS_COMPOSITOR_COMMIT=jkl012...
PLAYOS_SHELL_COMMIT=mno345...

WLROOTS_COMMIT=pqr678...
RAYLIB_COMMIT=stu901...
MESA_VERSION=24.1.0
```

Update `versions.lock` via:
```bash
scripts/update-versions.sh --component playos-compositor --commit abc123
```

---

## Buildroot Package Structure

Each PlayOS component has a Buildroot package under `br2-external/package/`:

```
package/playos-compositor/
├── Config.in           # Kconfig entry: BR2_PACKAGE_PLAYOS_COMPOSITOR
└── playos-compositor.mk

# playos-compositor.mk
PLAYOS_COMPOSITOR_VERSION = $(call read-file,$(BR2_EXTERNAL_PLAYOS_PATH)/../../versions.lock,PLAYOS_COMPOSITOR_COMMIT)
PLAYOS_COMPOSITOR_SITE = https://github.com/PlayOS-Foundation/playos-compositor
PLAYOS_COMPOSITOR_SITE_METHOD = git
PLAYOS_COMPOSITOR_DEPENDENCIES = wlroots libdrm wayland wayland-protocols libxkbcommon
PLAYOS_COMPOSITOR_INSTALL_TARGET = YES

define PLAYOS_COMPOSITOR_BUILD_CMDS
    $(TARGET_MAKE_ENV) cmake -S $(@D) -B $(@D)/build \
        -DCMAKE_TOOLCHAIN_FILE=$(HOST_DIR)/share/buildroot/toolchainfile.cmake \
        -DCMAKE_BUILD_TYPE=Release
    $(TARGET_MAKE_ENV) cmake --build $(@D)/build
endef

define PLAYOS_COMPOSITOR_INSTALL_TARGET_CMDS
    $(INSTALL) -D -m 0755 $(@D)/build/playos-compositor $(TARGET_DIR)/usr/bin/playos-compositor
endef

$(eval $(generic-package))
```

---

## Image Variants

### Development image
- Includes: BusyBox, debug tools (`strace`, `gdbserver`, `evtest`, `modetest`)
- Serial console enabled
- Extra debug symbols
- Triggered by: `make qemu-config` or `make ally-config` (default)

### Production image
- No interactive shell
- No debug tools
- Signed EFI artifact
- Bounded logs
- Triggered by: `make ally-config PLAYOS_PROD=1` or via the release pipeline

### Installer image
- Contains: `playos-installer` Raylib UI + disk partitioning tools (`fdisk`, `mkfs.ext4`, `mkfs.fat`)
- EFI artifact wraps installer init, not the normal `playos-init`
- Triggered by: `make installer-image TARGET=rog-ally`

---

## Post-Build Artifacts

After `make qemu-build`:
```
build/qemu/
├── images/
│   ├── bzImage                    Linux kernel (fast dev)
│   ├── rootfs.cpio.zst            Initramfs (fast dev)
│   └── playos-esp.img             UEFI-bootable ESP image
└── staging/                       Sysroot for cross-development
```

After `make ally-build`:
```
build/rog-ally/
└── images/
    ├── playos-rog-ally-<version>-dev.img     Dev image
    ├── playos-rog-ally-<version>-prod.img    Production image
    └── playos-rog-ally-<version>-installer.img
```

---

## Build Time Estimates

| Target | Machine | First build | Incremental |
|---|---|---|---|
| QEMU | 8-core workstation | ~45 min | ~2 min |
| ROG Ally | 8-core workstation | ~60 min | ~5 min |
| QEMU | 4-core laptop | ~90 min | ~5 min |

Buildroot caches downloads in `~/.buildroot-dl/` (or `BR2_DL_DIR`). Sharing this directory across builds saves significant time.

---

## Forking Buildroot

**Do not fork Buildroot** unless a required change cannot be expressed as:
- An external package (`br2-external/package/`)
- A board configuration
- A kernel or package patch (`br2-external/patches/`)
- A rootfs overlay
- A post-build or post-image script

If a fork is truly required, document the reason as an ADR.
