# Reference Devices

**Reference devices** are specific hardware targets that the PlayOS
project actively maintains and tests against. They serve as the baseline
for porting and certification.

## What reference devices are

A reference device is a real piece of hardware with a maintained device
profile, input mapping, and backend implementation. It is not an abstract
category — it is a specific device you can buy and test on.

## Current reference devices

| Device | Type | Architecture | Primary input | Status |
|---|---|---|---|---|
| **ROG Ally** | Handheld PC | x86_64 | Built-in gamepad + touch | Active |
| **Generic Linux PC** | Desktop/laptop | x86_64 | Any controller | Active |
| **Orange Pi** | ARM SBC | aarch64 | Any controller | Planned |

## Why reference devices

- **Porting baseline** — if you can port to the ROG Ally, you can port to
  any x86_64 handheld. If you can port to the Orange Pi, you can port to
  any ARM SBC.
- **Certification reference** — the certification test suite runs on
  reference devices. A new device is compared against the reference.
- **Developer target** — developers can buy the same hardware the core
  team uses and get identical behavior.

## Relationship to device profiles

Each reference device has a device profile in
[`playos-reference-devices`](https://github.com/PlayOS-Foundation/playos-reference-devices)
that declares its capabilities, input mapping, and display properties.
This profile is the starting point for porting to similar hardware.

## Adding a reference device

1. Propose the device via RFC (or issue).
2. Write a device profile.
3. Implement or adapt the Platform API backends.
4. Add to the CI/test matrix.
5. Document in this chapter.

## Related

- [ROG Ally Reference](../12-device-model-and-porting/12-rog-ally-reference.md)
- [Generic Linux PC Reference](../12-device-model-and-porting/13-generic-linux-pc-reference.md)
- [Orange Pi Reference](../12-device-model-and-porting/14-orange-pi-reference.md)
- [Certified Devices](05-certified-devices.md)
