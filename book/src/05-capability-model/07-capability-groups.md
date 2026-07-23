# Capability Groups

Capabilities are organized into **groups** by their top-level namespace.
Groups keep the registry navigable and help developers understand which
part of the platform a capability belongs to.

## Group list

| Group | Namespace | Examples | Description |
|---|---|---|---|
| **Input** | `input.*` | `input.basic`, `input.touch`, `input.gyroscope`, `input.accelerometer` | Input devices and sensors |
| **Display** | `display.*` | `display.info`, `display.hdr`, `display.high-refresh` | Display properties and modes |
| **Storage** | `storage.*` | `storage.local`, `storage.external` | File system access |
| **Audio** | `audio.*` | `audio.playback`, `audio.microphone`, `audio.spatial` | Audio input and output |
| **Power** | `power.*` | `power.battery`, `power.performance-modes`, `power.thermal` | Power state and management |
| **Network** | `network.*` | `network.wifi`, `network.cellular`, `network.ethernet` | Network connectivity |
| **Cloud** | `cloud.*` | `cloud.saves`, `cloud.leaderboards`, `cloud.achievements` | Cloud-backed services |
| **System** | `system.*` | `system.overlay`, `system.game-switching`, `system.suspend-resume`, `system.package-install` | Runtime system features |
| **Platform** | `platform.*` | `platform.marketplace`, `platform.notifications` | Platform-level services |
| **Vendor** | `vendor.*` | `vendor.rog-ally.fan-control`, `vendor.rog-ally.tdp-control` | Device-specific extensions |

## Group membership rules

- Every capability belongs to exactly one group.
- The group is encoded in the capability identifier as the first segment
  before the dot.
- New groups may be added by RFC. Adding a group changes the registry
  schema.

## Why groups matter

Groups serve three purposes:

1. **Organization** — a flat list of 100+ capabilities is hard to navigate;
   groups provide structure.
2. **Documentation** — each Platform API module maps to one or more groups,
   making it clear which APIs depend on which capabilities.
3. **Tooling** — the SDK and certification tools can filter and validate
   capabilities by group.

## Related

- [Capability Registry](10-capability-registry.md)
- [Vendor Capabilities](09-vendor-capabilities.md)
- `schemas/capability-registry.schema.json`
