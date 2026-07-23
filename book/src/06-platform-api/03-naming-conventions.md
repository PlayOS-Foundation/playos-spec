# Naming Conventions

Every public symbol in the Platform API follows a consistent naming scheme.
This keeps the surface predictable and easy to learn.

## Namespaces

All public symbols live under `namespace PlayOS`. Modules use a nested
namespace matching the module name:

```cpp
namespace PlayOS {
namespace Input {            // module: input
namespace Capabilities {     // module: capabilities
namespace Lifecycle {        // module: lifecycle
namespace Storage {          // module: storage
namespace Display {          // module: display
namespace Audio {            // module: audio
namespace Network {          // module: network
namespace Battery {          // module: battery
namespace Touch {            // module: touch
namespace Bluetooth {        // module: bluetooth
```

Module names are PascalCase. The namespace is a direct child of `PlayOS`.

## Headers

- One public header per module: `include/playos/<module>.h`
  (e.g. `playos/input.h`, `playos/storage.h`).
- The umbrella header `include/playos/playos.h` includes every module header
  and defines the platform version.
- Header names are lowercase, matching the module name.
- Internal headers (backend interfaces, implementation details) live in
  `src/backends/` and are never exposed publicly.

## Functions

- Functions are PascalCase (matching C++ standard library style for
  stateless free functions): `Init()`, `Update()`, `Shutdown()`, `Down()`,
  `Pressed()`, `GetAxis()`, `Current()`, `SavePath()`, `PrimaryIP()`.
- Property accessors use `Get` prefix for non-trivial queries
  (`GetMasterVolume()`) or the bare property name for simple returns
  (`Current()`).
- Predicates use `Is` or `Has` prefix: `IsCharging()`, `IsMuted()`,
  `Has(cap)`.
- Mutators use `Set` prefix: `SetMuted()`, `SetMasterVolume()`.

## Enumerations

- Enumerations use `enum class` with PascalCase names:
  `Button`, `Axis`, `Capability`.
- Enumerators are PascalCase: `Button::A`, `Button::DPadUp`,
  `Capability::InputBasic`, `Capability::Touch`.
- Compound names merge without underscores: `DPadUp` not `D_Pad_Up`,
  `LeftX` not `Left_X`.
- A `Count` sentinel is placed last where array indexing is needed.

## Structs and types

- Data structures are simple `struct` with public members, PascalCase:
  `DisplayInfo`, `TouchPoint`, `WiFiNetwork`.
- Member fields are camelCase: `refreshRate`, `safeAreaLeft`,
  `primaryIP`.
- Types that own resources or have behavior use `class`, PascalCase:
  `DeviceProfile`.

## Capability identifiers

- Capability string identifiers (returned by `Capabilities::Id()`) use
  lowercase dot-separated namespace form: `input.touch`,
  `cloud.saves`, `system.overlay`.
- The first segment is the capability group; the second is the specific
  feature.
- Hyphens are used for multi-word segments: `suspend-resume`.

## Version

The platform version is a `MAJOR.MINOR.PATCH` triple defined as
preprocessor macros in `playos/playos.h`:

```cpp
#define PLAYOS_VERSION_MAJOR 0
#define PLAYOS_VERSION_MINOR 1
#define PLAYOS_VERSION_PATCH 0
```

## Related

- [API Design Rules](02-api-design-rules.md)
- [API Versioning](25-api-versioning.md)
- `include/playos/playos.h` (reference implementation)
