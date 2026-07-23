# Future Device Checklist

Use this checklist when proposing a new Runtime Device or SDK Target
for PlayOS. It ensures all the necessary artifacts exist before a
device is considered supported.

## Before proposing: capability assessment

| Question | Answer |
|---|---|
| Does the device run Linux (or can it be made to)? | |
| Is there a working GPU with DRM/KMS or EGL drivers? | |
| Are input devices exposed as evdev, XInput, or equivalent? | |
| Is audio available via PipeWire, PulseAudio, ALSA, or equivalent? | |
| Is there a network interface (Wi-Fi, Ethernet)? | |
| Does the device have persistent writable storage? | |
| Is there a working cross-compilation toolchain (if ARM/other arch)? | |

## Required artifacts

For a **Runtime Device**:

- [ ] Device profile (`device-profile.toml`) per [Device Profile Schema](02-device-profile-schema.md)
- [ ] Input backend implementing `IInputBackend`
- [ ] Display backend implementing `IDisplayBackend`
- [ ] Storage backend implementing `IStorageBackend`
- [ ] Audio backend implementing `IAudioBackend` (or null if no audio)
- [ ] Battery backend implementing `IBatteryBackend` (or null if no battery)
- [ ] Network backend implementing `INetworkBackend` (or null if no network)
- [ ] Bluetooth backend implementing `IBluetoothBackend` (or null if no Bluetooth)
- [ ] Touch backend implementing `ITouchBackend` (or null if no touch)
- [ ] CMake build preset
- [ ] Conformance suite passing (all registered capabilities)
- [ ] Device chapter in this part of the book

For an **SDK Target**:

- [ ] Input backend (at minimum)
- [ ] Display backend (at minimum)
- [ ] Storage backend (at minimum)
- [ ] Null backends for all other modules
- [ ] CMake build preset (including cross-compilation toolchain if needed)
- [ ] Device chapter in this part of the book
- [ ] Conformance suite passing (subset for present capabilities)

## Device profile checklist

```toml
[device]
id = ""          # Unique kebab-case identifier
name = ""        # Human-readable name
targetType = ""  # "runtime-device" or "sdk-target"

[input]          # Complete for all physical/primary inputs
[display]        # Width, height, refresh rate; safe areas if needed
[capabilities]   # Every capability the device provides

# Optional sections:
[quirks]         # Device-specific workarounds (vendor extensions)
```

## Design questions to answer

Before submitting, write answers to these:

1. **What is this device's primary use case?** (gaming handheld,
   development desktop, SBC kiosk, console, etc.)
2. **What existing PlayOS backend code can be reused?** (e.g., Linux
   evdev backend for any Linux device)
3. **What new backend code is needed?** List each module.
4. **What vendor extensions are needed?** (e.g., rear touchpad on
   Vita, TDP control on ROG Ally)
5. **What is the conformance threshold?** (Required vs optional
   capabilities)
6. **Is this a Runtime Device, SDK Target, or both?**
7. **What is the testing plan?** (Hardware availability, CI
   integration possibility)

## Coordination points

- **Spec change:** Does this device require changes to the Platform
  API, capability registry, or backend interfaces? If yes, write an
  RFC.
- **ADR needed:** Does this device introduce a new architectural
  assumption (new CPU arch, new kernel, new graphics API)? If yes,
  write an ADR.
- **Sibling repos:** Which sibling repositories need changes?
  (`playos-platform-api`, `playos-runtime`, `playos-shell`,
  `playos-reference-devices`, `playos-refdistro`)

## Example: proposing a new device

```text
Device: Steam Deck OLED
Type: Runtime Device
CPU: AMD Zen 2 (x86_64)
GPU: AMD RDNA 2 (amdgpu + Mesa)
Display: 1280×800, 90 Hz, HDR OLED
Input: Built-in gamepad (evdev), touchscreen, trackpads
Audio: PipeWire via ACP
Network: Wi-Fi 6E

Backends to reuse: All Linux backends (input, display, storage, network, bluetooth)
Backends to implement: HDR display info (extend IDisplayBackend), trackpad mapping
Vendor extensions: steam.hdr, steam.trackpad
New capability: display.hdr (needs RFC)
Conformance: Full suite (Runtime Device)
Testing: Physical hardware available for developers

Chapters to update:
- New: 12-device-model-and-porting/19-steam-deck-reference.md
- Update: 05-capability-model/10-capability-registry.md (add display.hdr)
```

## Related

- [Runtime Device Bringup](10-runtime-device-bringup.md)
- [SDK Target Bringup](11-sdk-target-bringup.md)
- [Backend Porting Model](03-backend-porting-model.md)
