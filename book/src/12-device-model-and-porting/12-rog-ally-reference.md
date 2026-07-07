# ROG Ally Reference

The **ASUS ROG Ally** is the primary reference runtime device for PlayOS and
the target for the vertical-slice proof-of-concept.

## Hardware summary

| Aspect | Detail |
|---|---|
| GPU | AMD Radeon 780M (RDNA 3 iGPU) |
| Graphics stack | Mesa + `vulkan-radeon` / RADV |
| Display | 1080p, up to 120 Hz |
| Input | Built-in gamepad controls + touchscreen |
| Vendor buttons | Armoury Crate button, Command Center button |

The 780M renders a Raylib shell at 1080p/120 Hz with negligible GPU usage,
leaving essentially all GPU headroom for games.

## Device profile (sketch)

```toml
[device]
id = "rog-ally"
name = "ASUS ROG Ally"
targetType = "runtime-device"

[input]
home_button = "asus_armoury"
quick_settings_button = "asus_command_center"
built_in_controls = "gamepad0"
touchscreen = "touch0"

[display]
default_width = 1920
default_height = 1080
refresh_rate = 120

[capabilities]
input.basic = true
input.touch = true
display.info = true
power.battery = true
display.brightness = true
system.overlay = true
```

Vendor buttons map to logical PlayOS inputs: **Armoury → Home**,
**Command Center → QuickSettings**. See
[Controller-First Navigation](../09-shell-and-ux/03-controller-first-navigation.md).

## GPU acceleration check

Confirm hardware rendering (not software):

```sh
glxinfo | grep renderer
# Expect: AMD Radeon 780M (or similar)
# If it shows llvmpipe, the GPU is not being used.
```

## Bring-up checklist (vertical slice)

```text
[ ] Arch installed; 780M brings up DRM/KMS (not llvmpipe)
[ ] seatd grants seat/device access without root
[ ] TinyWL-based compositor starts on the TTY and owns the display
[ ] Raylib shell renders as a Wayland client at 1080p
[ ] Built-in gamepad navigates the shell (A = launch, B = back)
[ ] Armoury button routes to Home (return-to-shell)
[ ] One demo game launches and, on exit/Home, returns to the shell
```

## Out of scope for the first slice

Touch input, brightness/TDP/fan control, suspend/resume, and audio are
deferred until the baseline console loop is proven.

See: [Boot Model](../08-runtime-architecture/03-boot-model.md),
[Compositor Model](../08-runtime-architecture/05-compositor-model.md),
[Runtime Devices](../04-target-model/02-runtime-devices.md).
