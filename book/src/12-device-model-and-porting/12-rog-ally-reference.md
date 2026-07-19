# ROG Ally Reference

The **ASUS ROG Ally** is the primary PlayOS Runtime Device and the hardware target for the Alpine reference-OS vertical slice.

## Hardware summary

| Aspect | Detail |
|---|---|
| CPU | AMD Ryzen Z1 / Z1 Extreme |
| GPU | AMD Radeon 780M-class RDNA 3 integrated graphics |
| Graphics stack | amdgpu + Mesa |
| Display | 1920×1080, up to 120 Hz |
| Input | Built-in gamepad, touchscreen, vendor buttons |
| Firmware | UEFI |
| Reference OS | Alpine Linux-based PlayOS image |

## Device policy

The device profile declares panel orientation, refresh modes, touch mapping, built-in controls, battery, brightness, and vendor-button capabilities.

- Armoury button → Home.
- Command Center button → Quick Settings.

These mappings belong to the device profile, not hard-coded shell behaviour.

## Bring-up modes

1. **Reference image:** boot the Alpine PlayOS image from `playos-refdistro`. This is authoritative.
2. **Development host:** build and launch the compositor and shell from Alpine installed on the Ally. This shortens development loops but does not replace image testing.

The former Arch/CachyOS host kit is legacy migration material.

## Hardware rendering

Bring-up must verify DRM and renderer state without relying on a desktop session:

```sh
readlink -f /sys/class/drm/card0/device/driver
cat /sys/kernel/debug/dri/0/name 2>/dev/null || true
```

Compositor logs must confirm amdgpu and a hardware EGL/GLES renderer rather than pixman or llvmpipe fallback.

## Alpine vertical-slice definition of done

```text
[ ] Alpine PlayOS image boots through UEFI
[ ] amdgpu firmware loads without missing-firmware errors
[ ] seatd grants compositor access without root
[ ] PlayOS compositor owns DRM/KMS
[ ] Raylib shell built against musl renders as a Wayland client
[ ] built-in gamepad navigates the shell
[ ] Home returns from a game to the existing shell
[ ] touchscreen maps to the internal panel
[ ] 60 Hz and 120 Hz modes are detected
[ ] sample game launches and returns cleanly
[ ] persistent PlayOS data survives reboot
[ ] first-frame timing is recorded
```

## Subsequent gates

- controller and dock hotplug;
- PipeWire audio;
- Wi-Fi and Bluetooth;
- brightness and battery;
- suspend/resume;
- external displays;
- recovery boot;
- signed system-image updates.

See [Boot Model](../08-runtime-architecture/03-boot-model.md), [Linux Reference Runtime](../08-runtime-architecture/04-linux-reference-runtime.md), and [ADR-0004](../../../adr/0004-use-alpine-linux-reference-os-base.md).
