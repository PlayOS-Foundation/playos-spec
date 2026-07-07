# Use Arch Linux as the Reference Runtime Base

- **ADR:** 0003
- **Title:** Use Arch Linux as the PlayOS Reference Runtime Base
- **Status:** Accepted
- **Date:** 2026-01-01
- **Deciders:** PlayOS Foundation

## Context

PlayOS Runtime Devices need a Linux-based operating system layer
beneath the compositor and shell. The base distribution must:

- be minimal (no desktop environment, no unnecessary services),
- support the latest kernel, Mesa, and GPU drivers for handheld
  gaming hardware (AMD Radeon 780M and similar),
- package wlroots, Wayland, libinput, PipeWire, and seatd,
- allow building a reproducible, custom ISO image,
- support x86_64 and ARM64 (Orange Pi / RK3588) architectures,
- be suitable for both prototyping and eventual production images.

## Decision

PlayOS will use **Arch Linux** as the reference runtime base, with a
phased approach:

1. **Prototype on real Arch** — install vanilla Arch on the ROG Ally,
   capture the exact package and configuration set.
2. **Freeze the working system** — export the package list
   (`pacman -Qqe`) and track all configuration files in Git.
3. **Package PlayOS software as Arch packages** — create `PKGBUILD`
   files for `playos-compositor`, `playos-shell`, `playos-services`,
   etc.
4. **Build a custom ISO with archiso** — use `mkarchiso` to produce a
   reproducible PlayOS ISO image.

The target boot sequence is:

```text
UEFI → Linux Kernel → systemd → seatd → PlayOS Compositor → PlayOS Shell
```

**Boot priority:** The critical path is GPU → input → shell. Audio
(PipeWire/WirePlumber), Wi-Fi, Bluetooth, and other services start
asynchronously and must never block the shell from appearing.

## Consequences

**Positive:**
- Arch provides the latest kernel, Mesa, and GPU drivers — critical
  for bleeding-edge handheld hardware.
- `archiso` is the official, well-documented tool for building custom
  Arch-based ISO images.
- `PKGBUILD` + `makepkg` provide a clean packaging workflow for
  PlayOS components.
- ARM64 support (ALARM / Arch Linux ARM) exists for Orange Pi and
  similar SBCs.
- Rolling release avoids major version upgrade disruptions.

**Negative:**
- Rolling release means upstream changes can break the build; CI must
  test regularly.
- Arch has a smaller ARM64 ecosystem than Debian/Ubuntu; some SBC
  bring-up may require additional effort.
- Not all users or OEMs are comfortable with Arch; downstream
  derivatives may choose different bases (the spec remains
  distribution-agnostic; this ADR covers only the reference
  implementation).

## Alternatives Considered

- **Debian/Ubuntu:** More stable and widely supported, but packages
  lag behind for GPU drivers and wlroots versions needed for modern
  handheld hardware.
- **Fedora:** Good balance of freshness and stability, but `archiso`
  is more mature and better documented than Fedora's image tools for
  this use case.
- **Buildroot / Yocto:** Maximal control and minimal footprint, but
  significantly more effort for a first prototype. May be appropriate
  for future embedded/minimal builds.
- **Bazzite / ChimeraOS / HoloISO:** These are existing gaming-focused
  distributions. Rejected because PlayOS needs to own its full stack
  rather than being a derivative of another gaming distro.
