# Runtime Devices

A **Runtime Device** runs the full PlayOS environment: the PlayOS Runtime,
Shell, package execution layer, marketplace client, system services,
compositor, and hardware integration.

## What a runtime device provides

A runtime device is expected to offer the complete console experience:

- boots directly into the PlayOS Shell,
- runs the compositor that owns the display,
- installs and executes PlayOS packages,
- provides system overlay, game switching, and suspend/resume,
- exposes device capabilities such as power, brightness, and touch where
  the hardware supports them.

## Reference Runtime Devices

The following are adopted as reference runtime devices:

- **ROG Ally** — the primary handheld reference (AMD Radeon 780M).
- **Generic x86_64 PC** — the baseline desktop/mini-PC target.
- **Orange Pi 5 (RK3588, ARM64)** — the low-cost ARM reference console.

Each proves a different part of the platform: vendor buttons and handheld
power management (ROG Ally), broad hardware variety (generic PC), and
ARM64 portability (Orange Pi).

## Boot model

The reference runtime boots directly into the console experience:

```text
UEFI → Linux Kernel → systemd → seatd → PlayOS Compositor → PlayOS Shell
```

The critical boot path is **GPU → input → shell**. Audio, Wi-Fi,
Bluetooth, library scanning, and cloud sync start asynchronously and must
never block the shell from appearing. See
[Boot Model](../08-runtime-architecture/03-boot-model.md).

## Relationship to SDK Targets

Runtime devices are a superset of [SDK Targets](03-sdk-targets.md): they
support the full Platform API plus runtime-only capabilities such as
system overlay, package installation, and OS-level suspend/resume.

Which features an application may use on a given device is determined by
the [Capability Model](../05-capability-model/01-overview.md), not by
assuming every runtime device is identical.
