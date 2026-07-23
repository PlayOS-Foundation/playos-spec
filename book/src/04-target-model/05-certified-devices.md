# Certified Devices

A **certified device** is hardware that has passed the PlayOS
certification test suite and is authorized to display the PlayOS
compatibility mark.

## Certification levels

| Level | Requirements | Mark |
|---|---|---|
| **Platform API Compatible** | Implements all required capabilities. Passes API conformance tests. | "PlayOS Compatible" |
| **Runtime Compatible** | Runs the full PlayOS Runtime. Supports overlay, game switching, package installation. | "PlayOS Runtime" |
| **PlayOS Certified Hardware** | Runtime Compatible + performance requirements, accessibility, security review. | "PlayOS Certified" |

## What certification guarantees

For players:
- The device runs PlayOS applications correctly.
- Input is mapped correctly to logical buttons.
- Performance meets minimum requirements.
- Security features are enabled.

For developers:
- A certified device behaves predictably.
- One binary works on all certified devices at the same level.
- The device profile accurately reflects the hardware.

## Certification process

1. **Submit** — OEM submits device for certification with a device
   profile and test results.
2. **Test** — PlayOS Foundation (or authorized lab) runs the
   certification test suite.
3. **Review** — Security and accessibility review (for Certified
   Hardware level).
4. **Certify** — Device is listed in the certified device registry.
5. **Maintain** — OEM must re-certify for major Runtime updates.

## Self-certification

For "Platform API Compatible" level, self-certification is accepted:
the OEM runs the conformance test suite and submits results. Higher
levels require third-party testing.

## Related

- [Certification](../14-certification/01-overview.md)
- [Reference Devices](04-reference-devices.md)
- [Device Profiles](07-device-profiles.md)
