---
rfc: 0014
title: Device Porting and Bring-Up Model
status: Draft
authors: []
created: 2026-07-22
---

# RFC-0014: Device Porting and Bring-Up Model

> **Status:** Draft

## Summary

This RFC defines the process and requirements for porting PlayOS to a
new hardware target: what constitutes a port, the backend interface
model, the bring-up checklist for Runtime Devices and SDK Targets, and
how new devices join the PlayOS ecosystem.  It provides the foundation
for Part XII (Device Model and Porting) of the PlayOS Book.

## Motivation

PlayOS targets diverse hardware — handheld PCs, single-board computers,
desktops, and mobile devices.  Without a standardised porting model:

- Each port would reinvent the backend interface.
- Bring-up would have no checklist; critical steps would be missed.
- Device profiles would be inconsistent.
- The certification process would have no reference for what "ported"
  means.

A formal porting model ensures every device gets the same quality of
integration and every port follows the same process.

## Guide-Level Explanation

Porting PlayOS to a new device means:

1. **Write a device profile** — a TOML file describing the hardware.
2. **Implement missing backends** — if the device has hardware that no
   existing backend supports, write a new backend.
3. **Follow the bring-up checklist** — verify each subsystem in order.
4. **Submit for certification** — if the device is targeting PlayOS
   Certified Hardware.

For many devices, step 2 is empty — the existing Linux evdev, amdgpu,
and ALSA backends cover most PC hardware.  A Raspberry Pi needs a new
GPIO backend; a custom handheld needs a new input mapping.

## Reference-Level Explanation

### 1. Porting Levels

| Level | Work Required | Example |
|---|---|---|
| **Profile-only port** | Write a device profile. All hardware is covered by existing backends. | Generic Linux PC, most x86 handhelds |
| **Backend port** | Write one or more new backends for unsupported hardware. | Orange Pi (GPIO, Mali GPU), PS Vita (custom input) |
| **OS port** | Port the full Runtime + compositor to a new operating system kernel. | Future: FreeBSD, Redox |

Most devices are profile-only ports.  Backend ports are needed when the
device has hardware without an existing Linux driver or when a new OS
platform is targeted.  OS ports are rare — the reference implementation
targets Linux; Windows and macOS are SDK Targets that don't need the
full Runtime.

### 2. Device Profile (Step 1)

Every port starts with a device profile.  The format is defined by
RFC-0006.  The profile declares:

```toml
[device]
id = "my-device"
name = "My Device"
targetType = "runtime-device"  # or "sdk-target"

[input]
home_button = "xbox_guide"
built_in_controls = "gamepad0"

[display]
default_width = 1920
default_height = 1080
refresh_rate = 60

[capabilities]
"input.basic" = true
"input.touch" = true
"display.brightness" = true
"power.battery" = true
```

The profile is the single source of truth for what the device supports.
The capability model (RFC-0003) queries the profile at runtime — no
recompilation is needed to add or remove capabilities.

### 3. Backend Interface Model (Step 2)

When hardware is not covered by an existing backend, a new backend is
written.  Each backend implements a C++ interface defined by the
Platform API:

| Backend Interface | Purpose | Default Implementation |
|---|---|---|
| `IInputBackend` | Read gamepad, keyboard, touch, sensors | evdev (Linux), XInput (Windows) |
| `IDisplayBackend` | Query resolution, refresh rate, safe area | DRM (Linux), Win32 (Windows) |
| `IAudioBackend` | Enumerate devices, control volume | ALSA/PipeWire (Linux), WASAPI (Windows) |
| `IPowerBackend` | Battery status, TDP, brightness | sysfs (Linux), Win32 (Windows) |
| `IStorageBackend` | Resolve save/config paths | XDG (Linux), KnownFolders (Windows) |
| `INetworkBackend` | Enumerate interfaces, signal strength | NetworkManager/rtnetlink (Linux) |

#### Backend discovery

Backends are discovered through a **backend registry**:

```cpp
// In the Platform API initialization:
void RegisterBackends() {
    // Try each backend in priority order
    #if defined(__linux__)
        if (EvdevInputBackend::IsAvailable())
            Input::SetBackend(std::make_unique<EvdevInputBackend>());
        else
    #endif
        Input::SetBackend(std::make_unique<NullInputBackend>());
}
```

The backend registry tries platform-native backends first, then falls
back to Raylib/SDL polling backends, then to null backends (graceful
no-op).

#### Backend requirements

A backend MUST:

- Implement the full interface — no partial implementations.
- Return capability-appropriate errors for unsupported features
  (not crash or return garbage).
- Be selectable at compile time (CMake option) and at runtime
  (backend registry).
- Not introduce `#ifdef` in public headers — platform detection is
  inside the backend `.cpp` file.

### 4. Runtime Device Bring-Up Checklist

The bring-up proceeds in phases.  Each phase must be complete before
the next begins.

#### Phase 1: Boot and Display

```
[ ] Device boots to Linux kernel
[ ] GPU driver loads (verify: DRM card present, renderer is hw)
[ ] Framebuffer console works
[ ] seatd grants compositor access
[ ] PlayOS compositor owns DRM/KMS
[ ] Compositor renders at native resolution
[ ] Refresh rate matches device profile
```

#### Phase 2: Input

```
[ ] Built-in gamepad detected by evdev
[ ] All buttons report correct key codes
[ ] Touchscreen maps to internal panel
[ ] Home button and QuickSettings button mapped per profile
[ ] Shell navigable with built-in controls
[ ] External controller hotplug works
```

#### Phase 3: Shell and Game Loop

```
[ ] Shell renders as Wayland client
[ ] Library displays installed applications
[ ] Sample game launches from Shell
[ ] Game exits and Shell returns to foreground
[ ] Home button returns from game to Shell
[ ] First-frame timing meets target
```

#### Phase 4: Platform Services

```
[ ] Audio output works (PipeWire/ALSA)
[ ] Wi-Fi connects and survives suspend
[ ] Bluetooth pairs controllers
[ ] Battery status reported correctly
[ ] Brightness control works
[ ] Persistent data survives reboot
```

#### Phase 5: Certification Readiness

```
[ ] Suspend/resume works
[ ] External display hotplug
[ ] Controller and dock hotplug
[ ] Recovery boot works
[ ] Verified boot chain intact
[ ] All ACTS tests pass
[ ] All RCTS tests pass
```

### 5. SDK Target Bring-Up Checklist

SDK Targets are simpler — no compositor, no overlay, no package
installation.  The checklist is:

```
[ ] Platform API compiles and links on target OS
[ ] Input backend works (native or polling)
[ ] Storage backend resolves correct paths
[ ] Display backend reports correct resolution
[ ] Lifecycle::Init/Update/Shutdown work correctly
[ ] Capabilities report correctly for the target
[ ] ACTS test suite passes
[ ] Sample app (hello-playos) runs correctly
[ ] Input bridging works (if engine integration kit used)
```

### 6. Port Deliverables

A completed port MUST include:

| Deliverable | Runtime Device | SDK Target |
|---|---|---|
| Device profile | ✅ | ✅ |
| Backend implementations (if needed) | ✅ | ✅ |
| Bring-up checklist (completed) | ✅ | ✅ |
| ACTS test report | ✅ | ✅ |
| RCTS test report | ✅ | ❌ |
| Boot timing report | ✅ | ❌ |
| Known limitations document | ✅ | ✅ |

### 7. Reference Ports

The PlayOS project maintains reference ports that serve as examples and
test targets:

| Port | Type | Status |
|---|---|---|
| ROG Ally | Runtime Device | In progress (vertical slice) |
| Generic Linux PC | Runtime Device | Planned |
| Orange Pi | Runtime Device | Planned |
| Windows | SDK Target | Planned |
| Android | SDK Target | Planned |
| PS Vita | SDK Target | Planned |

### 8. Adding a New Port

1. Create directory: `playos-reference-devices/<device-id>/`.
2. Write `device-profile.toml`.
3. Implement any missing backends in `playos-platform-api/src/backends/`.
4. Follow the bring-up checklist.
5. Submit a PR with the profile, backends, and completed checklist.
6. Device is listed in the PlayOS Device Registry.

## Drawbacks

- The bring-up checklist is long — devices with unusual hardware may
  take weeks to fully bring up.
- Requiring full backend interfaces even for devices that only support
  a subset means writing null implementations for unsupported features.
- The backend registry model (try A, fall back to B, fall back to null)
  adds complexity to Platform API initialization.

## Rationale and Alternatives

- **Per-device #ifdef instead of backends:** Simpler but violates the
  capability-based principle (RFC-0001, Principle 4) and makes the
  codebase unmaintainable as devices multiply.
- **Single monolithic backend that supports everything:** Impractical —
  the backend would need to know about every device's hardware.
  Interface-based backends keep device-specific code isolated.
- **Auto-generated device profiles from hardware probing:** Attractive
  but unreliable — probing cannot detect capabilities that are absent
  (e.g., "does this device have a gyroscope?").  Manual profiles are
  the source of truth.
- **No checklist; ad-hoc bring-up:** Rejected — the checklist ensures
  consistent quality across ports and prevents missed steps.

## Prior Art

- **Linux kernel `arch/` and `drivers/`:** The backend interface model
  is directly inspired by the kernel's driver model — each piece of
  hardware gets a driver that implements a common interface.
- **Android HAL (Hardware Abstraction Layer):** Android's HAL defines
  interfaces for each hardware subsystem; device makers implement them.
  PlayOS backends serve the same role.
- **Yocto Project BSP (Board Support Package):** Yocto's BSP layers add
  support for new hardware.  PlayOS device profiles serve a similar
  purpose at a higher level of abstraction.
- **SteamOS / ChimeraOS hardware support:** Both use standard Linux
  drivers + a device-specific configuration layer.  PlayOS follows this
  pattern with TOML profiles replacing bespoke config files.

## Unresolved Questions

- How are device-specific kernel patches managed?  The reference OS is
  Alpine; kernel patches go through aports.  This is an OS-level
  concern, not a Platform API concern.
- Should device profiles include kernel command-line parameters?
- How are firmware blobs distributed for devices that require them?
  The reference OS must include necessary firmware in the image.

## Future Possibilities

- **Community port registry:** A public registry of community-maintained
  device profiles and backends, curated by the Foundation.
- **Porting wizard:** A CLI tool (`playos-port-wizard`) that guides new
  porters through the checklist interactively.
- **Automated bring-up testing:** CI that runs ACTS on each new device
  profile against the reference backends.
- **Hardware abstraction for non-Linux OSes:** Formalise the OS port
  process for FreeBSD or Redox, defining which backends need new
  implementations.
