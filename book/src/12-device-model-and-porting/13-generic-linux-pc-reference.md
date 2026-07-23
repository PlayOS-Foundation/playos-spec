# Generic Linux PC Reference

The **Generic Desktop / VM** profile is the baseline Runtime Device
for keyboard-and-mouse setups. It represents any standard x86_64
Linux desktop, laptop, or virtual machine.

## Purpose

This profile is the **lowest-common-denominator device** — it
defines the minimal expectations for a PlayOS Runtime Device without
a built-in gamepad, battery, or touchscreen.

## Hardware assumptions

| Aspect | Detail |
|---|---|
| Input | USB/Bluetooth keyboard and mouse |
| Display | Standard monitor, 1920×1080 at 60 Hz |
| Audio | Onboard or USB audio (PipeWire/PulseAudio/ALSA) |
| Network | Wired Ethernet or USB Wi-Fi |
| GPU | Any GPU with working DRM/KMS or EGL |

## Device profile

```toml
[device]
id = "generic-desktop"
name = "Generic Desktop / VM"
targetType = "runtime-device"

[input]
keyboard = "keyboard0"
mouse = "mouse0"
built_in_controls = "none"

[display]
default_width = 1920
default_height = 1080
refresh_rate = 60

[capabilities]
"input.basic" = true
"input.keyboard" = true
"input.mouse" = true
"display.info" = true
```

## Capabilities

| Capability | Status | Notes |
|---|---|---|
| `input.basic` | ✅ Required | Keyboard + mouse input |
| `input.keyboard` | ✅ Present | Standard HID keyboard |
| `input.mouse` | ✅ Present | Standard HID mouse |
| `display.info` | ✅ Required | Monitor via EDID/DRM |
| `power.battery` | ❌ Absent | Desktop assumes AC power |
| `input.touch` | ❌ Absent | No touchscreen |
| `system.overlay` | ❌ Absent | No overlay hotkey by default |

## Backends used

All backends are from the `linux/` directory:

| Module | Backend | Status |
|---|---|---|
| Input | `linux_input_backend.cpp` | Keyboard + mouse only (no gamepad by default) |
| Display | `linux_display_backend.cpp` | ✅ |
| Storage | `linux_storage_backend.cpp` | ✅ |
| Audio | Null (future: PipeWire) | ❌ Missing |
| Battery | Null (no battery) | N/A |
| Network | `linux_network_backend.cpp` | ✅ |
| Touch | Null (no touch) | N/A |
| Bluetooth | `linux_bluetooth_backend.cpp` | ✅ If adapter present |

## Testing on a VM

The Generic Desktop profile is the primary VM target:

1. Create a VM with VirtIO GPU (or virgl for 3D).
2. Boot the Alpine PlayOS image.
3. Verify the compositor starts on the VirtIO framebuffer.
4. Verify keyboard and mouse input via evdev.

## Related

- [ROG Ally Reference](12-rog-ally-reference.md) — handheld reference
- [Backend Porting Model](03-backend-porting-model.md)
- [Linux Reference Runtime](../08-runtime-architecture/04-linux-reference-runtime.md)
