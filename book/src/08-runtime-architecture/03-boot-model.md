# Boot Model

The PlayOS reference runtime boots directly into the console experience. No
login manager, no desktop environment, no visible terminal.

## Boot sequence

```text
UEFI → Linux Kernel → systemd → seatd → PlayOS Compositor → PlayOS Shell
```

- **UEFI / bootloader** hands off to the kernel.
- **Linux kernel** initializes the GPU (DRM/KMS) and input devices.
- **systemd** starts the PlayOS session as its default target.
- **seatd** provides seat and device access to the compositor without
  requiring root.
- **PlayOS Compositor** (wlroots/TinyWL) takes ownership of the display.
- **PlayOS Shell** starts as a Wayland client and draws the console UI.

## Critical path vs background

The **critical boot path** is intentionally minimal:

```text
GPU  →  input  →  shell
```

Only what is required to show an interactive UI is on the critical path.
Everything else starts **asynchronously** and MUST NOT block the shell from
appearing:

```text
Background (async):
  audio (PipeWire / WirePlumber)
  Wi-Fi
  Bluetooth
  battery / power monitor
  library scan
  updates
  cloud sync
```

The shell appears immediately and updates status indicators live as
background services become ready ("Wi-Fi: connecting…", "Library:
refreshing…").

## Timing targets

```text
v0.1 target:   UI visible in under 10 seconds
future target: UI visible in under 5 seconds
resume target: 1–2 seconds
```

## Requirements

- The runtime MUST reach an interactive shell using only GPU and input.
- Audio MUST NOT be a boot dependency; the shell MUST be usable while audio
  is still initializing (see [Audio Startup](13-audio-startup.md)).
- Non-critical services MUST start asynchronously.

## Vertical slice note

For the proof-of-concept, the minimal viable boot is: kernel brings up the
780M GPU and the Ally's input devices, `seatd` grants access, the compositor
starts on the TTY, and the Raylib shell renders. Audio, Wi-Fi, and library
scanning are out of scope for the first slice.

See: [Compositor Model](05-compositor-model.md),
[Linux Reference Runtime](04-linux-reference-runtime.md), and
[ADR-0003](https://github.com/PlayOS-Foundation/playos-spec/blob/main/adr/0003-arch-linux-reference-runtime-base.md).
