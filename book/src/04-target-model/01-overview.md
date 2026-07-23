# Target Model Overview

The **target model** defines what hardware and software environments
PlayOS runs on — and what developers need to know about each one.

## Two target categories

PlayOS distinguishes two fundamentally different target categories:

1. **Runtime Devices** — hardware that runs the full PlayOS Runtime.
   These are the "console" experience players see.

2. **SDK Targets** — development platforms that run PlayOS applications
   directly, without the full Runtime. These are what developers use
   for development, testing, and prototyping.

The distinction matters because the same application binary works on
both, but the runtime services available differ.

## Compatibility levels

Not all devices are equal. Three compatibility levels define what
developers can expect:

| Level | Required capabilities | Runtime services | Certification |
|---|---|---|---|
| Platform API Compatible | ✅ | ❌ | Self-test |
| Runtime Compatible | ✅ | ✅ | Optional |
| PlayOS Certified Hardware | ✅ | ✅ | Required |

For details, see [Target Compatibility Levels](08-target-compatibility-levels.md).

## Reference devices

The project maintains a set of **reference devices** — specific
hardware that serves as the porting and certification baseline. See
[Reference Devices](04-reference-devices.md).

## Key concepts

- Runtime Devices and SDK Targets are **not the same thing**.
- Applications query capabilities at runtime, never check which target
  they are on.
- Device profiles declare what a device can do.
- Platform backends implement the API on each OS.
- Certified devices meet higher bars for performance, accessibility,
  and security.

## Chapter index

| Chapter | Description |
|---|---|
| [Runtime Devices](02-runtime-devices.md) | Hardware that runs the full Runtime |
| [SDK Targets](03-sdk-targets.md) | Development platforms |
| [Reference Devices](04-reference-devices.md) | Specific hardware baselines |
| [Certified Devices](05-certified-devices.md) | Certification levels and process |
| [Platform Backends](06-platform-backends.md) | OS-specific API implementations |
| [Device Profiles](07-device-profiles.md) | TOML files that describe hardware |
| [Target Compatibility Levels](08-target-compatibility-levels.md) | What each level means |
