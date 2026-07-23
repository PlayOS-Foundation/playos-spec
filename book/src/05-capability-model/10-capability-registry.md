# Capability Registry

The **capability registry** is the authoritative list of every capability
recognized by the PlayOS platform. It defines the identifier, group, status,
and description of each capability.

## Schema

The registry is defined by
[`capability-registry.schema.json`](../../schemas/capability-registry.schema.json).
Each entry has:

| Field | Type | Description |
|---|---|---|
| `id` | string | Stable namespaced identifier, e.g. `input.touch` |
| `status` | `required` \| `optional` \| `vendor` \| `deprecated` | Category |
| `group` | string | Capability group, e.g. `input` |
| `summary` | string | Short human-readable description |

## Current registry

The following capabilities are defined in the Platform API `Capability` enum
(`playos/capabilities.h`) and listed here in registry order.

### Required

| Identifier | Group | Description |
|---|---|---|
| `input.basic` | input | Digital buttons and at least one analog stick via logical PlayOS buttons |
| `storage.local` | storage | Read/write access to save and config paths |
| `display.info` | display | Query display size, refresh rate, and safe area |
| `lifecycle.basic` | lifecycle | Initialization, update loop, and shutdown |

### Optional

| Identifier | Group | Description |
|---|---|---|
| `input.touch` | input | Multi-touch input (finger position and count) |
| `power.battery` | power | Battery level and charging state |
| `display.brightness` | display | Display backlight brightness control |
| `network.info` | network | WiFi state, IP address, network scan and connect |
| `bluetooth.present` | system | Bluetooth host controller presence |
| `cloud.saves` | cloud | Cloud-backed save synchronization |
| `system.overlay` | system | System overlay surface |
| `platform.marketplace` | platform | Store client and entitlement API |
| `system.suspend-resume` | system | Application suspend and resume |
| `cloud.achievements` | cloud | Achievement unlock and progress |
| `cloud.leaderboards` | cloud | Leaderboard submission and query |
| `cloud.accounts` | cloud | Player account sign-in and identity |
| `system.game-switching` | system | Switch between running applications without restart |
| `system.package-install` | system | Install, verify, update, and remove .gpk packages |
| `platform.notifications` | platform | System and application notification delivery |

## Adding a capability

1. Propose the capability in an RFC, including the identifier, group, status,
   and the API surface it guards.
2. Add the entry to this chapter and to the
   `Capability` enum in `playos/capabilities.h`.
3. Update `capabilities.cpp` to register the capability at startup.
4. At least one reference backend must implement the corresponding API
   surface.

## Deprecating a capability

A capability marked `deprecated` in the registry remains in the enum
(IDs are never removed, to preserve binary compatibility) but reports
absent on all new targets. Deprecated capabilities follow the
[Deprecation Policy](../16-governance-and-process/08-deprecation-policy.md).

## Related

- [Capability Groups](07-capability-groups.md)
- [Vendor Capabilities](09-vendor-capabilities.md)
- `schemas/capability-registry.schema.json`
- `playos/capabilities.h` (Platform API header)
