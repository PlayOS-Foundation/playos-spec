# Capabilities API

The `PlayOS::Capabilities` module lets applications discover what the current
device can do at runtime. It is the mechanism that makes the
[Capability Model](../05-capability-model/01-overview.md) executable.

## API

```cpp
namespace PlayOS {
namespace Capabilities {

bool Has(Capability capability);           // Is a capability present?
std::vector<Capability> List();            // Which capabilities are present?
const char* Id(Capability capability);     // Stable namespaced identifier

} // namespace Capabilities
} // namespace PlayOS
```

## Enumerated capabilities

The `Capability` enum lists every capability known to this Platform API
version. Each entry has a stable string identifier (for example
`Capability::InputBasic` → `"input.basic"`) returned by `Id()`.

Required capabilities (always present on conformant targets):
`InputBasic`, `StorageLocal`, `DisplayInfo`, `LifecycleBasic`.

Optional capabilities (query before use): `Touch`, `Battery`, `Brightness`,
`CloudSave`, `Overlay`, `Marketplace`, `Suspend`, `Achievements`,
`Leaderboards`.

## Usage

```cpp
if (PlayOS::Capabilities::Has(PlayOS::Capability::Touch)) {
    auto touches = PlayOS::Input::TouchPoints();
} else {
    // Use controller-only navigation.
}
```

## Where capabilities come from

The set of present capabilities is provided by the device profile and backend
at startup. Required capabilities are always present on conformant targets;
optional capabilities depend on hardware and the target type (Runtime Device
vs SDK Target).

## Requirements

- `Has(capability)` MUST return `true` for every required capability.
- `Has` for an optional capability MUST reflect actual device availability.
- `List()` MUST include exactly the capabilities the device provides.
- `Id()` MUST return a stable, namespaced string (dot-separated, lowercase,
  for example `"input.touch"`) that matches the
  [Capability Registry](../99-appendices/02-capability-registry.md).
- The capability set is read-only after `Init`; it does not change at
  runtime.
