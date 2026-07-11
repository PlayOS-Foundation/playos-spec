# Linux Reference Runtime

The PlayOS Linux reference runtime is the concrete implementation of the
runtime contract for Linux-based devices such as the ASUS ROG Ally.  It is
built from a small set of well-established Linux components and distributed
as a bootable ISO image produced by `playos-refdistro`.

## Components

| Component | Technology | Role |
|---|---|---|
| Base OS | Arch Linux | Package ecosystem, rolling updates, minimal base |
| Init system | systemd | Service ordering, targets, socket activation |
| Display | wlroots 0.19 + DRM/KMS | Compositor and page-flip engine |
| Renderer | EGL/GLES2 (hardware), pixman (virtual GPU) | GPU-adaptive rendering |
| Input | libinput + evdev | Keyboard, pointer, gamepad |
| Audio | PipeWire + WirePlumber | Session audio routing |
| Network | NetworkManager + iwd | WiFi, Ethernet |
| Bluetooth | BlueZ | Controller pairing |
| Seat management | seatd | DRM/input device access without root |
| Shell | `playos-shell` (Raylib, Wayland client) | Game library and launcher |
| Package filesystem | EROFS (compressed, read-only) + tmpfs overlay | Fast mount, writable layer |

## Boot sequence

```
UEFI firmware
  └─ GRUB / systemd-boot
      └─ Linux kernel (vmlinuz)
          └─ initramfs (archiso hook, overlay mount)
              └─ systemd PID 1
                  ├─ local-fs.target
                  ├─ systemd-udevd  (GPU + input enumeration)
                  ├─ seatd          (seat management)
                  ├─ playos-compositor-env  (GPU detection → /run/playos/compositor.env)
                  ├─ playos-compositor  (DRM takeover, launches shell)
                  │     └─ playos-shell  (first frame visible ≈ 2 s)
                  └─ playos-async.target  (audio, network, BT, library, updates)
```

Target: first shell frame visible in **≤ 2 seconds** from UEFI handoff.

## GPU renderer selection

The runtime auto-detects the primary DRM driver at boot:

```bash
# /usr/bin/playos-compositor-env  (runs as ExecStartPre)
DRV=$(basename $(readlink -f /sys/class/drm/card0/device/driver))
case "$DRV" in
    vmwgfx|virtio_gpu|qxl|bochs)
        echo "WLR_RENDERER=pixman" > /run/playos/compositor.env ;;
esac
```

| Driver | Renderer selected | Notes |
|---|---|---|
| `amdgpu` | EGL/GLES2 | ROG Ally, desktop AMD GPUs |
| `i915`, `xe` | EGL/GLES2 | Intel integrated / Arc |
| `nouveau`, `nvidia` | EGL/GLES2 | Nvidia |
| `vmwgfx` | pixman (software) | VMware Workstation |
| `virtio_gpu` | pixman (software) | QEMU/KVM virtio |
| `qxl`, `bochs` | pixman (software) | Other virtual GPUs |

## Filesystem layout

```
/                      erofs read-only image (LZMA compressed)
  /usr/bin/            runtime binaries (compositor, shell, tools)
  /playos-samples/     pre-installed sample games
  /run/playos/         runtime state (tmpfs, writable)
    wayland-0          Wayland socket
    compositor.env     GPU-adaptive renderer override
  /var/lib/playos/     persistent application data (tmpfs overlay, 512 MB)
  /var/log/playos/     runtime logs
```

The writable overlay is a `tmpfs` mount provisioned by the archiso
`cow_spacesize=512M` kernel parameter.  All writes (save data, config,
logs) go to this overlay; the base EROFS image is never modified at
runtime.

## Systemd targets

```
playos-visual.target   (default.target)
  requires: seatd, playos-compositor
  wants: playos-firstboot

playos-async.target
  wants: playos-audio, playos-network, playos-bluetooth,
         playos-library, playos-update
```

`playos-compositor.service` MUST NOT have a `Requires=` or `After=` on any
async service.  Async services MAY have `After=playos-compositor.service`
but MUST NOT block it.

## Platform API wiring on Linux

| Module | Backend | Source |
|---|---|---|
| Input (gamepad) | `linux_input_backend` (evdev) | `/dev/input/event*` |
| Input (keyboard) | Wayland seat via compositor | routed to Wayland client |
| Storage | `linux_storage_backend` | XDG Base Dir (`~/.local/share/PlayOS`) |
| Display | `raylib_display_backend` | Raylib monitor query |
| Audio | `raylib_audio_backend` | Raylib / miniaudio |
| Lifecycle | `raylib_lifecycle_backend` | Raylib window events |
| Capabilities | runtime-configured | `storage.local`, `input.basic`, `display.info` |

## Reference device: ASUS ROG Ally

| Property | Value |
|---|---|
| CPU | AMD Ryzen Z1 Extreme |
| GPU | AMD Radeon 780M (RDNA 3, amdgpu driver) |
| Display | 1920×1080, 120 Hz |
| Input | Built-in controller (Xbox layout), touchscreen |
| Boot firmware | UEFI, Secure Boot optional |
| PlayOS renderer | EGL/GLES2 (hardware accelerated) |

The ROG Ally bring-up kit is in `playos-reference-devices/rog-ally/`.
It provides a setup script, build script, and session launcher for
developers working directly on the device without the full ISO.

## Known limitations (v0.1)

| Area | Status |
|---|---|
| Overlay UI | Not yet implemented; Home button handled in shell |
| Suspend/resume | Not yet implemented |
| Persistent storage across boots | tmpfs — all writes lost on reboot |
| Controller hotplug | First detected gamepad only; no hotplug |
| Audio | PipeWire installed but not wired through Platform API |
| Pointer/touch input | libinput receives events; Wayland routing not connected |}
