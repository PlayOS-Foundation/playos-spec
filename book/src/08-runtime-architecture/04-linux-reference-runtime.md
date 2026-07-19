# Linux Reference Runtime

The PlayOS Linux reference runtime implements the runtime contract for Linux Runtime Devices such as the ASUS ROG Ally. It is integrated into a bootable reference OS produced by `playos-refdistro`.

The reference OS uses Alpine Linux. Alpine is an implementation choice, not part of the portable PlayOS Platform API. Other conforming runtimes may use other operating-system bases.

## Components

| Component | Technology | Role |
|---|---|---|
| Base OS | Pinned Alpine stable | Minimal appliance userland |
| C library | musl | Native reference ABI |
| Package manager | apk | Image and component packaging |
| Init system | OpenRC | Boot ordering and service lifecycle |
| Image builder | Alpine aports + mkimage | Reproducible image profiles |
| Display | wlroots + DRM/KMS | Compositor infrastructure |
| Renderer | EGL/GLES2; pixman fallback | Hardware and virtual rendering |
| Input | libinput + evdev | Keyboard, pointer, touch, gamepad |
| Audio | PipeWire + WirePlumber | Audio routing |
| Network | NetworkManager + iwd | Wi-Fi and Ethernet |
| Bluetooth | BlueZ | Controllers and accessories |
| Seat management | seatd | Unprivileged device access |
| Shell | `playos-shell` | Raylib Wayland client |
| Boot filesystem | Read-only Alpine image + modloop/initramfs | Replaceable system image |
| Persistent state | Dedicated PlayOS data partition | Saves, configuration, logs, packages |

## Boot sequence

```text
UEFI
  → bootloader
  → Linux kernel
  → Alpine initramfs and read-only root
  → OpenRC sysinit/boot
  → GPU and input readiness
  → seatd
  → playos-compositor
      → playos-shell
      → compositor readiness
  → playos-async runlevel
      → audio, network, Bluetooth, library, updates
```

The visual runlevel must not depend on network, Bluetooth, audio, cloud, marketplace, update, or indexing readiness.

The target remains a first interactive shell frame within two seconds of UEFI handoff on the reference device. This is a measured target, not an assumption guaranteed by changing distributions.

## OpenRC service model

The reference image defines:

- `playos-visual`: device readiness, seatd, compositor, and shell;
- `playos-async`: audio, networking, Bluetooth, indexing, updates, cloud, marketplace, and debug access.

Long-running services should use OpenRC supervision and explicit readiness where available. The compositor permits the asynchronous runlevel only after the Wayland socket exists and the shell can present.

No asynchronous service may be a dependency of `playos-compositor`.

## Image and persistence model

The ISO follows Alpine read-only image, initramfs, and modloop conventions. Development images may use Alpine diskless mode.

Installed Runtime Devices use a separate writable PlayOS data partition:

```text
/var/lib/playos/       application and runtime state
/var/log/playos/       retained diagnostics
/var/cache/playos/     package and update cache
/home/playos/          shell-local state where required
```

The system image must be replaceable without deleting player saves. The final update design may use signed A/B images. Diskless `apkovl` and local APK caches are development and recovery mechanisms, not substitutes for PlayOS package/update contracts.

## Native and compatibility runtimes

The compositor, shell, runtime services, and reference Platform API backend must compile against musl.

Portable applications should consume PlayOS contracts rather than the host C library. Packages needing glibc, Steam Runtime, Wine/Proton, or another userspace must declare and enter an explicit compatibility runtime.

The host may provide `gcompat` for limited cases, but arbitrary glibc compatibility is not guaranteed.

## GPU renderer selection

The runtime may choose a safe fallback based on the primary DRM driver:

```sh
driver="$(basename "$(readlink -f /sys/class/drm/card0/device/driver)")"
case "$driver" in
    vmwgfx|virtio_gpu|qxl|bochs)
        export WLR_RENDERER=pixman
        ;;
esac
```

| Driver | Renderer | Notes |
|---|---|---|
| `amdgpu` | EGL/GLES2 | ROG Ally and AMD GPUs |
| `i915`, `xe` | EGL/GLES2 | Intel GPUs |
| `nouveau`, `nvidia` | EGL/GLES2 where supported | NVIDIA hardware |
| `vmwgfx`, `virtio_gpu`, `qxl`, `bochs` | pixman fallback | Virtual bring-up |

Renderer selection must be validated rather than inferred solely from the driver name.

## Platform API wiring

| Module | Backend | Source |
|---|---|---|
| Input | Linux input backend and Wayland seat | evdev and compositor routing |
| Storage | Linux storage backend | PlayOS persistent data root |
| Display | Wayland/runtime display backend | compositor output state |
| Audio | Linux audio backend | PipeWire |
| Lifecycle | PlayOS Runtime | supervised application state |
| Capabilities | runtime and device profile | declared and probed capabilities |

## Reference device

The ASUS ROG Ally is the first hardware reference: AMD Ryzen Z1-class CPU, Radeon 780M-class GPU, 1920×1080 120 Hz panel, built-in controller, touchscreen, and UEFI firmware.

The Alpine kit lives in `playos-reference-devices/rog-ally/`.

## Alpine validation gates

The Alpine image is not feature-equivalent to the legacy Arch prototype until it validates:

- UEFI and recovery boot;
- amdgpu firmware and hardware rendering;
- wlroots compositor and Raylib shell on musl;
- built-in controls, Home, touch, and hotplug;
- 60/120 Hz output policy and panel orientation;
- audio, Wi-Fi, Bluetooth, and dock networking;
- suspend/resume and external displays;
- persistent data across system updates;
- crash recovery and return to shell;
- repeated cold-boot and first-frame measurements.

See [ADR-0004](../../../adr/0004-use-alpine-linux-reference-os-base.md).
