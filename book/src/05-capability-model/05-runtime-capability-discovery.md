# Runtime Capability Discovery

Applications discover capabilities **at runtime** through the Platform API,
not at compile time and not by inspecting the platform.

## The API

```cpp
// Is a capability present on this device right now?
bool hasTouch = PlayOS::Capabilities::Has(PlayOS::Capability::Touch);

// Enumerate everything this device provides.
for (auto cap : PlayOS::Capabilities::List()) {
    // inspect cap
}
```

- `Has(capability)` returns whether a specific capability is available.
- `List()` enumerates all present capabilities.

## Typical pattern

```cpp
if (PlayOS::Capabilities::Has(PlayOS::Capability::Touch)) {
    auto touches = PlayOS::Input::TouchPoints();
    // use touch
} else {
    // fall back to controller navigation
}
```

## Where capabilities come from

```text
Device Profile (declares capabilities)
   ↓
Backend (implements the corresponding API surface)
   ↓
Capabilities::Has / List (reports them to the application)
```

The device profile declares what the hardware offers; the backend implements
it; the capability API reports it. Adding a device therefore means writing a
profile (and backend where needed), not changing application code.

## Rules

- Availability MUST be discoverable at runtime via `Has`/`List`.
- Applications MUST query optional capabilities before using them.
- Discovery MUST be cheap enough to call during normal execution.
- Reported capabilities MUST reflect the actual device, not a compile-time
  assumption.

## Vertical slice note

The proof-of-concept can ship a minimal capability provider that reports the
required set (`input.basic`, `storage.local`, `display.info`,
`lifecycle.basic`) plus the runtime capabilities the slice uses. The shell
queries nothing beyond what it needs, keeping the first implementation small.

See: [Overview](01-overview.md),
[Required Capabilities](03-required-capabilities.md),
[Capabilities API](../06-platform-api/08-capabilities-api.md).
