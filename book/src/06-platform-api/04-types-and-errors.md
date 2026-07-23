# Types and Errors

The Platform API uses a minimal, portable type system. There is no universal
`Result<T, E>` — each module documents its own error behavior per the
[capability failure rules](../05-capability-model/06-capability-failure-behaviour.md).

## Primitive types

The Platform API uses C++ standard types only:

| Type | Use |
|---|---|
| `bool` | Predicates and initialization success |
| `int` | Counts, pixel positions, refresh rate |
| `float` | Analog values (axes, volume, battery level) |
| `const char*` | Paths, IP addresses, capability identifiers |
| `std::string` | Owned strings (network SSIDs, device names) |
| `std::vector<T>` | Lists (capabilities, touch points, WiFi networks) |

No custom integer types, no platform-dependent typedefs. The API is designed
to be usable from C-compatible FFI boundaries in the future, which is why
`const char*` is preferred over `std::string` for return values that are
pointer-stable.

## Structures

Data aggregates are plain `struct` with public members:

```cpp
struct DisplayInfo {
    int width;
    int height;
    int refreshRate;
    int safeAreaLeft, safeAreaTop, safeAreaRight, safeAreaBottom;
};

struct TouchPoint {
    int id;
    int x;
    int y;
};
```

Structures are value types — copyable, assignable, and cheap to pass by
value or const reference. No inheritance, no virtual methods at the public
API boundary.

## Enumerations

All enumerations use `enum class` for type safety:

```cpp
enum class Button { A, B, X, Y, /* ... */ Count };
enum class Capability { InputBasic, StorageLocal, /* ... */ };
```

Enum values are contiguous integers starting at zero where the implementation
needs array indexing (e.g. `Button`, `Axis`). A `Count` sentinel is placed
last.

## Error handling

There is no universal error type. Each module uses the simplest mechanism
appropriate to its semantics:

| Mechanism | When used | Example |
|---|---|---|
| **Return `bool`** | Initialization, simple success/failure | `Lifecycle::Init()` returns `false` on failure |
| **Sentinel value** | Informational queries where a default exists | `Battery::Level()` returns `-1.0` when capability absent |
| **Return `enum` result** | Operations with multiple distinct outcomes | `Network::Connect()` returns `ConnectResult::Success` / `WrongPassword` / `Timeout` / `Error` |
| **No-op / empty result** | Fire-and-forget or list queries | `ScanNetworks()` returns empty vector when no hardware |
| **Documented undefined** | Cheap getters where caller is expected to guard | `Touch::GetPoint(index)` — undefined if `index >= PointCount()` |

## Capability-gated failures

Functions gated by an optional capability must fail predictably when the
capability is absent. The failure mode is documented per-function (see the
[Capability Failure Behaviour](../05-capability-model/06-capability-failure-behaviour.md)
chapter for the three modes: error code, no-op, and fallback).

## Requirements

- Public API MUST NOT throw exceptions across module boundaries.
- Public API MUST NOT return raw owning pointers (`new`/`delete`).
- Every function MUST document its behavior when preconditions are violated
  or the required capability is absent.
- Structures MUST be trivially copyable where the implementation stores them
  in arrays.

## Related

- [API Design Rules](02-api-design-rules.md)
- [Capability Failure Behaviour](../05-capability-model/06-capability-failure-behaviour.md)
- [Naming Conventions](03-naming-conventions.md)
