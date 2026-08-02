# PlayOS Developer Environment Guide

> **Cross-references:** [build-guide.md](build-guide.md), [testing.md](testing.md), [Sprint-0.md](Sprint-0.md)

---

## Development Workflow Overview

```
┌─────────────────────────────────────────────────────────┐
│  Fast iteration path (QEMU)                             │
│                                                         │
│  Edit code → make qemu-build → make qemu-run → verify  │
│  (~2 min for incremental builds)                        │
└─────────────────────────────────────────────────────────┘
          │ Physical validation
          ▼
┌─────────────────────────────────────────────────────────┐
│  Hardware validation (ROG Ally)                         │
│                                                         │
│  make ally-build → flash USB → boot on Ally → verify   │
│  (Required for: GPU, input, audio, ACPI, thermal)       │
└─────────────────────────────────────────────────────────┘
```

**Rule:** QEMU covers boot logic, process lifecycle, IPC, and storage. Physical hardware is required for AMDGPU, controller input, audio, power management, and display behavior.

---

## QEMU Setup

### OVMF (UEFI firmware for QEMU)

```bash
# Ubuntu
sudo apt-get install -y ovmf

# Fedora
sudo dnf install -y edk2-ovmf

# Verify
ls /usr/share/OVMF/OVMF_CODE.fd   # or /usr/share/edk2/x64/OVMF.fd on Fedora
```

### `make qemu-run` (what it does)

```bash
#!/bin/bash
# scripts/run-qemu.sh
BUILD=$1

qemu-system-x86_64 \
  -machine q35 \
  -cpu host \
  -enable-kvm \
  -m 4G \
  -smp 4 \
  -bios /usr/share/OVMF/OVMF_CODE.fd \
  -drive if=pflash,format=raw,readonly=on,file=/usr/share/OVMF/OVMF_VARS.fd \
  -drive file=${BUILD}/images/playos-esp.img,format=raw,if=none,id=esp \
  -device nvme,drive=esp,serial=playos-esp \
  -drive file=${BUILD}/images/data.img,format=raw,if=none,id=data \
  -device nvme,drive=data,serial=playos-data \
  -device virtio-gpu \
  -display sdl,gl=on \
  -audiodev pa,id=audio0 \
  -device intel-hda \
  -device hda-duplex,audiodev=audio0 \
  -serial stdio \
  -device virtio-net-pci,netdev=net0 \
  -netdev user,id=net0 \
  -no-reboot
```

**Important:** Always boot via OVMF (not `-kernel`). The UEFI boot path must be validated, not bypassed.

### QEMU virtual data disk

Create a virtual data disk for the `/data` partition:

```bash
# Create 8 GB virtual disk
make qemu-create-data-disk    # or:
qemu-img create -f raw build/qemu/images/data.img 8G
```

This disk is formatted by `playos-init` on first boot (provisioning mode with `playos.mode=provision` on the kernel cmdline for testing).

### Headless QEMU (CI)

```bash
qemu-system-x86_64 \
  -machine q35 -cpu qemu64 -m 2G \
  -bios /usr/share/OVMF/OVMF_CODE.fd \
  ... \
  -display none \
  -serial file:serial.log \
  -no-reboot \
  -device isa-debug-exit,iobase=0xf4,iosize=0x04
```

The test harness sends a debug exit command via the QEMU monitor when the boot check passes.

---

## Nested Wayland Compositor (Developer Desktop)

For compositor development on a developer workstation (without QEMU overhead):

```bash
# Set backend to nested Wayland
export PLAYOS_BACKEND=nested
export WAYLAND_DISPLAY=wayland-0  # your host compositor

./build/playos-compositor &

# In another terminal:
export WAYLAND_DISPLAY=playos-0
./build/playos-shell
```

The compositor runs nested inside your desktop session. No root or DRM access needed.

For headless compositor testing on CI:
```bash
export PLAYOS_BACKEND=headless
./build/playos-compositor
```

---

## Cross-Compilation

Buildroot produces a cross-toolchain at `build/rog-ally/host/`. You can use it directly to build and test individual components:

```bash
export PATH="$PWD/build/rog-ally/host/bin:$PATH"
export PKG_CONFIG_PATH="$PWD/build/rog-ally/staging/usr/lib/pkgconfig"
export PKG_CONFIG_LIBDIR="$PWD/build/rog-ally/staging/usr/lib/pkgconfig"
export CC=x86_64-buildroot-linux-musl-gcc
export CXX=x86_64-buildroot-linux-musl-g++

# Build a component against the staging sysroot
cmake -S playos-compositor -B build-cross \
  -DCMAKE_TOOLCHAIN_FILE=$PWD/build/rog-ally/host/share/buildroot/toolchainfile.cmake \
  -DCMAKE_BUILD_TYPE=Debug
cmake --build build-cross
```

---

## USB Boot Workflow (ROG Ally)

```bash
# Build the USB image
make ally-build
make ally-usb-image

# Flash to USB (replace /dev/sdX with your USB device)
sudo dd if=build/rog-ally/images/playos-rog-ally-dev.img of=/dev/sdX bs=4M status=progress oflag=sync

# Boot the Ally from USB:
# 1. Hold Volume Down while pressing Power
# 2. Select USB device in UEFI boot menu
# 3. PlayOS boots from USB
```

---

## Serial Console

Serial output is the primary debug log during bring-up. The ROG Ally has no physical serial port, but there are two options:

### Option 1: USB serial adapter (hardware)
Connect a USB-to-serial adapter to the ROG Ally's USB-C port using a custom cable or dock with serial passthrough. Use `minicom` or `picocom` on the host.

### Option 2: kernel earlycon + netconsole (software)
For early boot output, configure the kernel command line:
```
console=ttyS0,115200 earlycon=serial8250,io,0x3f8,115200
```

For post-boot network logging (requires networking):
```
netconsole=@/,@<host-ip>/
```

### QEMU serial
QEMU serial output goes to stdout (`-serial stdio`). For CI, redirect to a file (`-serial file:serial.log`).

---

## Debugging Tools (Development Image)

Available in the development image (not production):

| Tool | Purpose |
|---|---|
| `evtest` | Test input devices — verify controller button mapping |
| `modetest` | Test DRM/KMS — verify connectors, encoders, CRTCs |
| `strace` | Trace system calls of any process |
| `gdbserver` | Remote GDB debugging (requires networking) |
| `perf` | CPU/GPU performance profiling |
| `aplay` / `arecord` | ALSA audio testing |
| `weston-info` | Inspect Wayland compositor info |
| BusyBox shell | Available at `/bin/sh` — not present in production |

Access the development shell:
- QEMU: serial console (available from boot)
- ROG Ally: cannot access in normal mode — reboot with `playos.shell=1` kernel param (development only)

---

## Iterative Development Loop

### Modifying `playos-compositor`:
```bash
# 1. Edit source in playos-compositor/
vim playos-compositor/src/compositor.c

# 2. Rebuild just that package in Buildroot
make -C buildroot O=build/qemu playos-compositor-rebuild

# 3. Re-run QEMU
make qemu-run
```

### Modifying `playos-shell`:
```bash
make -C buildroot O=build/qemu playos-shell-rebuild
make qemu-run
```

### Modifying the kernel:
```bash
make -C buildroot O=build/qemu linux-rebuild
make qemu-run
# Kernel rebuilds are slower (~5 min)
```

### Working on `playos-platform-api` (host native build):
```bash
# Build and test on the host for faster iteration
cmake -S playos-platform-api -B build-host -DPLAYOS_BACKEND=stub
cmake --build build-host
ctest --test-dir build-host
```

---

## Environment Variables Reference

| Variable | Used by | Purpose |
|---|---|---|
| `PLAYOS_BACKEND` | `libplayos`, compositor | Select backend: `drm`, `headless`, `nested`, `stub` |
| `WAYLAND_DISPLAY` | All Wayland clients | Which Wayland socket to connect to |
| `PLAYOS_TRUSTED_SHELL` | `playos-shell` | Marks client as trusted shell (`=1`) |
| `PLAYOS_TRUSTED_OVERLAY` | `playos-overlay` | Marks client as trusted overlay (`=1`) |
| `PLAYOS_LAUNCH_TOKEN` | Game process | One-time launch identity UUID |
| `PLAYOS_GAME_ID` | Game process | Game identifier string |
| `PLAYOS_INSTALL_PATH` | Game process | `/data/games/<id>` |
| `PLAYOS_SAVE_PATH` | Game process | `/data/saves/<id>` |
| `PLAYOS_CACHE_PATH` | Game process | `/data/cache/<id>` |
| `PLAYOS_LIFECYCLE_FD` | Game process | Read end of lifecycle pipe |
| `PLAYOS_COMPOSITOR_READY_FD` | `playos-compositor` | Write end — signal readiness to `playos-init` |
| `PLAYOS_AUDIO_DEVICE` | `libplayos` | Override ALSA device name for testing |
