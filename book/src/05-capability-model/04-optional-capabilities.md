# Optional Capabilities

**Optional capabilities** are features that may or may not be present on a
given PlayOS target. They are the reason the capability model exists:
without optional capabilities, every device would be identical.

## The query-before-use rule

An application MUST query an optional capability before using the API it
guards:

```cpp
if (PlayOS::Capabilities::Has(PlayOS::Capability::Touch)) {
    // safe to call TouchPoints()
    auto touches = PlayOS::Input::TouchPoints();
}
```

Calling an API guarded by an absent optional capability MUST return a
well-defined error or no-op, never crash (see
[Capability Failure Behaviour](06-capability-failure-behaviour.md)).

## Optional capability categories

| Category | Examples | Typically on |
|---|---|---|
| **Input** | `input.touch`, `input.gyroscope`, `input.accelerometer` | Handhelds, mobile |
| **Display** | `display.hdr`, `display.high-refresh` | Premium devices |
| **Audio** | `audio.spatial`, `audio.microphone` | Varies |
| **Power** | `power.battery`, `power.performance-modes` | Handhelds, laptops |
| **Network** | `network.cellular` | Mobile, some handhelds |
| **Cloud** | `cloud.saves`, `cloud.leaderboards`, `cloud.achievements` | Runtime Devices |
| **System** | `system.overlay`, `system.game-switching`, `system.suspend-resume`, `system.package-install` | Runtime Devices |
| **Platform** | `platform.marketplace`, `platform.notifications` | Runtime Devices |

## Which targets provide which

- **SDK Targets** (Windows, macOS, Linux desktop, Android, PS Vita): the
  core subset (required capabilities) plus whatever hardware the device
  has — touch on Android/PS Vita, battery on laptops and Android, gyro on
  PS Vita. Cloud and system capabilities are absent.
- **Runtime Devices**: the full set — core subset, hardware optional
  capabilities, cloud services, and system/runtime features.

## Runtime-only capabilities

Capabilities under `system.*` and `cloud.*` are **runtime-only** — they
require the PlayOS Runtime. An SDK Target reports them as absent. A
Runtime Device may still report some absent (e.g., a desktop runtime
device without a battery reports `power.battery` absent).

See the [Target Model](../04-target-model/01-overview.md) for the
distinction between SDK Targets and Runtime Devices.

## Adding an optional capability

1. Propose it via RFC against the
   [Capability Registry](../../schemas/capability-registry.schema.json).
2. Define the API it guards in the Platform API part.
3. Implement the backend for at least one reference target.

## Related

- [Required Capabilities](03-required-capabilities.md)
- [Capability Failure Behaviour](06-capability-failure-behaviour.md)
- [Vendor Capabilities](09-vendor-capabilities.md)
- RFC-0002 (Runtime Devices and SDK Targets)
- RFC-0003 (Capability Model)
