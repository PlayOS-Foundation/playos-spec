# Vendor Capabilities

**Vendor capabilities** are device- or vendor-specific extensions beyond the
standard capability set. They allow certified hardware to expose unique
features while keeping the core platform clean.

## Naming convention

Vendor capabilities use the `vendor.` namespace followed by the vendor
identifier:

```
vendor.<vendor-id>.<feature>
```

Examples:

| Capability | Device | Description |
|---|---|---|
| `vendor.rog-ally.fan-control` | ROG Ally | Adjust fan curves |
| `vendor.rog-ally.tdp-control` | ROG Ally | Set TDP limits per profile |
| `vendor.rog-ally.rgb-control` | ROG Ally | Control RGB lighting |
| `vendor.steam-deck.back-buttons` | Steam Deck | Extra grip buttons |

The vendor identifier must be a lowercase, hyphenated form of the vendor
or device name, registered in the capability registry.

## Rules

- Vendor capabilities MUST be optional. No device is required to support
  another vendor's extensions.
- Vendor capabilities MUST NOT duplicate a standard capability. If a
  feature belongs in the platform, propose it as a standard capability
  via RFC.
- Applications MUST guard vendor capabilities with both a capability
  query and a fallback path. Vendor features are never guaranteed.
- The capability registry records vendor capabilities but does not
  enforce them — they are informational for discovery and tooling.

## Certification impact

A device with vendor capabilities can still achieve certification. Vendor
capabilities are excluded from conformance testing — they are bonus
features, not requirements.

An application that depends on a vendor capability may not work on all
certified devices. This is acceptable; the application should declare the
dependency in its manifest and the marketplace may filter listings
accordingly.

## When to propose a standard capability instead

Ask: "Could this feature reasonably exist on multiple devices from
different vendors?" If yes, propose it as a standard optional capability
under the appropriate group. If the feature is genuinely tied to one
device's hardware, it is a vendor capability.

## Related

- [Capability Groups](07-capability-groups.md)
- [Capability Registry](10-capability-registry.md)
- [ROG Ally Reference](../12-device-model-and-porting/12-rog-ally-reference.md)
