# Overview

The **capability model** is how PlayOS applications discover what a device
can do. It is one of the defining features of the platform.

## The core rule

> Applications query capabilities. They do not check platforms.

Instead of branching on the operating system:

```cpp
#ifdef ANDROID
    // ...
#endif
```

applications ask what the current device supports:

```cpp
if (PlayOS::Capabilities::Has(PlayOS::Capability::Touch)) {
    auto touches = PlayOS::Input::TouchPoints();
}
```

## Why capabilities instead of platform checks

Devices differ. One device may support touch but not cloud saves. Another
may support cloud saves but not system overlay. Another may support the
Platform API but not the full runtime. Platform checks (`#ifdef`) hardcode
assumptions that break as soon as a new device appears.

Capabilities make these differences **explicit and discoverable**, so the
same binary can adapt to many devices at runtime.

## How capabilities relate to the target model

- [SDK Targets](../04-target-model/03-sdk-targets.md) expose the portable
  subset of capabilities.
- [Runtime Devices](../04-target-model/02-runtime-devices.md) add
  runtime-only capabilities such as overlay, package installation, and
  suspend/resume.
- Certified hardware may add vendor capabilities such as fan or TDP
  control.

## Categories of capability

Capabilities fall into groups specified later in this part:

- **Required** — capabilities every conformant target must provide (see
  [Required Capabilities](03-required-capabilities.md)).
- **Optional** — capabilities that may or may not be present (see
  [Optional Capabilities](04-optional-capabilities.md)).
- **Vendor** — device- or vendor-specific extensions (see
  [Vendor Capabilities](09-vendor-capabilities.md)).

## Failure behavior

When an application uses a capability that is not present, the platform must
fail predictably rather than crash. The rules are specified in
[Capability Failure Behaviour](06-capability-failure-behaviour.md).

## Future-proofing

Adding a new device should mean writing a device profile and, where needed,
a backend — never rewriting the SDK. The capability model is what makes
that possible. See the
[Device Model and Porting](../12-device-model-and-porting/01-overview.md)
part.
