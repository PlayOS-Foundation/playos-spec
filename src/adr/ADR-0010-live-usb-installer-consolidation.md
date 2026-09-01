# ADR-0010: Live USB Is The Installer (Sprint 13.7 Option B)

Status: Accepted (Sprint 13.7)

## Context

Earlier plans kept a separate "installer" image. The runtime installer handoff
(IPC + supervisor) made that unnecessary: the same live USB that boots the
session can install itself to the internal disk.

## Decision

- The **live USB image IS the installer**: Settings → Install PlayOS triggers
  a runtime handoff (stop shell/overlay/compositor/ssh, close `/data` log fds,
  preserve dev SSH key, unmount `/data` + `/EFI`, spawn the installer under
  supervision, reboot on exit).
- The installed kernel is the **same `bzImage`** used on the live ESP.
- A `playos-a` payload (squashfs rootfs + BOOTX64.EFI) is staged on the USB.
- An ESP marker (`/EFI/playos/live-usb`) plus removable-disk detection keeps
  live boots from pivoting into an installed internal slot.
- Headless automation uses `playos.install.auto` to drive the same path in QEMU.

## Consequences

- One image per target instead of separate live/installer images.
- The installer is always as fresh as the live system.
- Handoff complexity moved into `playos-init` supervisor, validated end-to-end
  in QEMU and on-device.
