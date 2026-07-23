# Orange Pi Reference

The **Orange Pi** family of ARM single-board computers is a reference
Runtime Device for ARM-based platforms. It proves PlayOS on non-x86
hardware — critical for expanding beyond PC handhelds.

## Purpose

The Orange Pi reference validates:
1. ARM CPU bringup (aarch64).
2. Mali/ARM GPU stack (Panfrost/Lima drivers).
3. Low-power embedded device constraints.
4. GPIO and HAT peripheral integration (future).

## Hardware assumptions

| Aspect | Detail |
|---|---|
| CPU | Allwinner H6 or Rockchip RK3588 (quad/octa-core Cortex-A) |
| GPU | ARM Mali (Panfrost driver or vendor blob) |
| RAM | 2–8 GB LPDDR4 |
| Storage | microSD or eMMC |
| Display | HDMI output, 1920×1080 at 60 Hz |
| Input | USB gamepad or GPIO-attached controller |
| Network | Onboard Wi-Fi + Ethernet |

## Target profile

```toml
[device]
id = "orange-pi"
name = "Orange Pi"
targetType = "runtime-device"

[display]
default_width = 1920
default_height = 1080
refresh_rate = 60

[capabilities]
"input.basic" = true
"display.info" = true
```

## Key challenges

### GPU driver stack

Unlike x86_64 (where amdgpu + Mesa is a known quantity), ARM GPUs
require the Panfrost (Mali G-series) or Lima (Mali 400/450) driver.
These are reverse-engineered Mesa drivers — they work well on
mainline kernels but may need device-tree configuration.

**Verification:** Ensure `panfrost.ko` or `lima.ko` loads, and
`glxinfo` reports a hardware renderer.

### Kernel and device tree

Orange Pi boards often ship with vendor kernels. Mainline kernel
support varies by model. The PlayOS reference image must target a
mainline or near-mainline kernel with working DRM/KMS, HDMI audio,
and Wi-Fi.

### Input options

Most Orange Pi boards have no built-in gamepad. Options:
- USB gamepad (evdev, same as desktop Linux).
- GPIO button matrix (requires a custom evdev driver or userspace daemon).
- Bluetooth gamepad via onboard Bluetooth.

### Storage wear

microSD cards have limited write endurance. The storage backend
should use a wear-leveled filesystem (F2FS) or redirect heavy writes
to a RAM overlay. This is a runtime concern, not a backend concern.

## Current status

| Component | Status | Notes |
|---|---|---|
| Kernel (mainline) | Varies by board | Orange Pi 5 (Rockchip) has good mainline |
| GPU (Panfrost) | Working on supported boards | Orange Pi 4/5 |
| Device profile | Draft | Needs board-specific input mapping |
| Backends | Not yet ported | Should reuse Linux backends where possible |
| PlayOS image | Not yet built | ARM Alpine exists; PlayOS layer needed |

## Bringup priorities

1. Boot Alpine aarch64 on Orange Pi 5.
2. Verify DRM/KMS with Panfrost.
3. Port Linux backends (most should work as-is on ARM).
4. Create device profile with correct input mappings.
5. Test with USB gamepad.
6. Add GPIO controller support (Stage 2).

## Related

- [ROG Ally Reference](12-rog-ally-reference.md) — x86_64 handheld reference
- [Backend Porting Model](03-backend-porting-model.md)
- [Input Porting](04-input-porting.md)
