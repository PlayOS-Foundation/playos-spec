# Capability Failure Behaviour

When an application uses a Platform API that depends on a capability the
current device does not provide, the platform must fail **predictably**.
Never crash, never leave undefined state, never silently succeed with
unexpected behavior.

## Failure modes

| Mode | When | Example |
|---|---|---|
| **Error code** | The call cannot proceed without the capability | `GetAxis(Axis::TouchX)` when `input.touch` is absent returns `Error::CapabilityNotAvailable` |
| **No-op** | The call is a request that can safely be ignored | `RequestNotification(...)` when `platform.notifications` is absent does nothing and returns success |
| **Fallback** | The platform provides a degraded but valid response | `GetBatteryLevel()` when `power.battery` is absent returns `100` (always plugged in) |

## Which mode to use

- **Error code** is the default. If the caller cannot get meaningful
  results without the capability, return an error.
- **No-op** applies to fire-and-forget requests where the caller does not
  depend on the outcome. Notifications and analytics submissions are
  typical examples.
- **Fallback** applies to informational queries where a sensible default
  exists. Battery level and display HDR support are typical examples.

Each API module documents the failure behavior for every function gated
by an optional capability.

## The application's responsibility

Applications are expected to guard optional capabilities with a query
before calling dependent APIs:

```cpp
if (PlayOS::Capabilities::Has(PlayOS::Capability::Touch)) {
    auto touches = PlayOS::Input::TouchPoints();  // safe
} else {
    // use controller navigation
}
```

Calling a guarded function without checking first is legal — the platform
will return an error — but it is not recommended. The error return ensures
the application can recover, but the user experience may degrade.

## Required capabilities never fail

Required capabilities are guaranteed present on every conformant target.
APIs that depend only on required capabilities never return
`CapabilityNotAvailable`. If a required capability is absent from a
device, that device is non-conformant and must not claim certification.

## Related

- [Optional Capabilities](04-optional-capabilities.md)
- [Required Capabilities](03-required-capabilities.md)
- RFC-0003 (Capability Model)
