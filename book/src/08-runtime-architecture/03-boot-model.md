# Boot Model

The PlayOS reference OS boots directly into the console experience. It has no display manager, desktop environment, or visible login session.

## Boot sequence

```text
UEFI
  → bootloader
  → Linux kernel and Alpine initramfs
  → OpenRC sysinit/boot
  → GPU and input readiness
  → seatd
  → PlayOS Compositor
  → PlayOS Shell first frame
  → asynchronous PlayOS services
```

OpenRC enters the PlayOS visual runlevel. seatd grants unprivileged device access. The compositor owns DRM/KMS, creates the Wayland socket, and starts the shell as a client. The asynchronous runlevel begins only after presentation readiness.

## Critical path

```text
read-only root
  → GPU and input
  → seat access
  → compositor
  → shell first frame
```

Audio, networking, Bluetooth, indexing, updates, cloud, marketplace, telemetry, and debug access must not block the first shell frame. The shell exposes live readiness states while they start.

## OpenRC runlevels

```text
playos-visual
  → required device readiness
  → seatd
  → playos-compositor
      → playos-shell

playos-async
  → audio
  → network
  → Bluetooth
  → library
  → updates
  → optional cloud and marketplace services
```

An asynchronous service may depend on compositor readiness. The compositor must not depend on an asynchronous service.

## Timing targets

| Milestone | v0.x acceptance | Reference target |
|---|---:|---:|
| Cold boot to first interactive shell | under 8 seconds | at most 2 seconds |
| Resume to interactive shell | under 3 seconds | at most 1 second |

Measurements begin at UEFI handoff and separate firmware, kernel, userspace, compositor, and first-frame time.

## Requirements

- The runtime must reach an interactive shell using only display, input, seat, and presentation.
- Audio and networking must not be boot dependencies.
- The first-frame path must not require remote services.
- Background services must start asynchronously.
- A failed background service must not terminate the compositor or shell.
- Home must remain available to the runtime even when a game is unresponsive.

## Vertical slice

The Alpine vertical slice is complete when the ROG Ally boots the read-only image, amdgpu provides hardware rendering, seatd grants access, the custom wlroots compositor owns DRM/KMS, and the Raylib shell presents an interactive frame.

See [Linux Reference Runtime](04-linux-reference-runtime.md), [Compositor Model](05-compositor-model.md), and [ADR-0004](../../../adr/0004-use-alpine-linux-reference-os-base.md).
