# Capability Based

PlayOS applications **query capabilities** at runtime rather than checking
the platform, operating system, or device name at compile time. This is the
mechanism that makes a single binary portable across many devices.

## The principle

> Applications ask what a device can do. They do not ask what the device is.

A handheld with touch, a desktop without, and a set-top box with neither
all run the same binary. The binary discovers at runtime which features
are available and adapts.

## Why capabilities

```cpp
// Wrong: platform check
#ifdef __ANDROID__
    EnableTouch();
#endif

// Right: capability query
if (PlayOS::Capabilities::Has(PlayOS::Capability::Touch)) {
    EnableTouch();
}
```

Platform checks (`#ifdef`) fail when a new device appears. They cannot
express that two devices on the same OS differ in hardware. They require
recompilation for each target. Capabilities solve all three problems.

## What capabilities cover

- **Hardware features** — touch, gyroscope, battery, brightness, haptics.
- **System features** — overlay, game switching, suspend/resume, package
  installation.
- **Cloud features** — cloud saves, leaderboards, achievements,
  marketplace.
- **Vendor features** — device-specific extensions (fan control, TDP, RGB).

## How capabilities are discovered

The application calls `Capabilities::Has()` or `Capabilities::List()`.
The result reflects the actual device, declared by its
[Device Profile](../12-device-model-and-porting/02-device-profile-schema.md)
and implemented by its backend.

## Related

- [Capability Model](../05-capability-model/01-overview.md)
- [Capabilities API](../06-platform-api/08-capabilities-api.md)
- [Specification First](01-specification-first.md)
