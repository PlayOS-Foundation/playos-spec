# Device Model and Porting Overview

This part describes how to **port PlayOS to new hardware** — adding
platform backends, writing device profiles, and bringing up new Runtime
Devices and SDK Targets.

## What this part covers

| Chapter | Audience |
|---|---|
| [Device Profile Schema](02-device-profile-schema.md) | All porters |
| [Backend Porting Model](03-backend-porting-model.md) | Backend implementors |
| [Input Porting](04-input-porting.md) | Input subsystem porters |
| [Display Porting](05-display-porting.md) | Display subsystem porters |
| [Audio Porting](06-audio-porting.md) | Audio subsystem porters |
| [Power Porting](07-power-porting.md) | Power/battery porters |
| [Storage Porting](08-storage-porting.md) | Storage subsystem porters |
| [Network Porting](09-network-porting.md) | Network subsystem porters |
| [Runtime Device Bringup](10-runtime-device-bringup.md) | Device integrators |
| [SDK Target Bringup](11-sdk-target-bringup.md) | SDK platform maintainers |
| Device-specific chapters (12–17) | Per-device porters |
| [Future Device Checklist](18-future-device-checklist.md) | Anyone adding a new device |

## How porting works

1. **Write a device profile** — a TOML file declaring the device's
   identity, input mappings, display properties, and capabilities.
2. **Implement backends** — for each Platform API module the device
   needs, write a backend class implementing the backend interface.
3. **Register in CMake** — add the backend source files to the build
   for the target platform.
4. **Test** — run the API conformance suite to verify correctness.

## Backend interfaces

Every Platform API module has an abstract backend interface in
`playos-platform-api/src/backends/`. Each interface has a single
responsibility and typically 2-5 methods. See [Backend Porting Model](
03-backend-porting-model.md) for details.

## Null backends as starting points

Every backend interface has a **null backend** — a conformant
implementation that reports capabilities absent and returns safe
defaults. Start porting by copying the null backend to a new
platform directory and implementing one method at a time.

## Reference devices

The project maintains profiles and backends for these reference
hardware targets:

| Device | Type | Backends |
|---|---|---|
| [ROG Ally](12-rog-ally-reference.md) | Runtime Device | linux/ |
| [Generic Linux PC](13-generic-linux-pc-reference.md) | Runtime Device | linux/ |
| [Orange Pi](14-orange-pi-reference.md) | Runtime Device | linux/ (planned) |
| [Windows](15-windows-sdk-target.md) | SDK Target | windows/ |
| [Android](16-android-sdk-target.md) | SDK Target | (planned) |
| [PS Vita](17-ps-vita-sdk-target.md) | SDK Target | (planned) |

## Related

- [Target Model](../04-target-model/01-overview.md)
- [Platform Backends](../04-target-model/06-platform-backends.md)
- [Engine Integration](../07-engine-integration/01-overview.md)
