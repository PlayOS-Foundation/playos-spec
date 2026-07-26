# API Versioning

The Platform API version is a `MAJOR.MINOR.PATCH` triple that follows
semantic-versioning principles adapted for a platform API surface.

## Version definition

```cpp
#define PLAYOS_VERSION_MAJOR 0
#define PLAYOS_VERSION_MINOR 1
#define PLAYOS_VERSION_PATCH 0
```

Applications can query the version at compile time via these macros,
defined in `playos/playos.h`.

## Versioning rules

### MAJOR (breaking changes)

Incremented when:
- A public symbol is removed or renamed.
- A public function signature changes incompatibly.
- An `enum class` enumerator is removed or reordered.
- A required capability is removed or changed to optional.

MAJOR versions may break source and binary compatibility. Multiple MAJOR
versions MAY coexist on the same device (side-by-side SDK installation).

### MINOR (additive changes)

Incremented when:
- A new module, function, enum, or capability is added.
- A new optional capability is introduced.
- A new backend is added.

MINOR versions are backward-compatible: code compiled against version
`0.N` links and runs against version `0.N+1` without recompilation.

### PATCH (fixes)

Incremented when:
- A bug fix does not change the public API surface.
- Documentation or tooling is updated.

PATCH versions are fully compatible.

## Compatibility window

The Platform API guarantees backward compatibility within a MAJOR version
for at least **two years** after the next MAJOR version is released. This
follows the [Long-Term Compatibility](../02-platform-principles/08-long-term-compatibility.md)
principle.

## Deprecation

Before removal, a public symbol MUST be marked deprecated for at least one
MINOR version. During the deprecation period:
- The symbol works normally.
- A compile-time deprecation warning is emitted.
- The deprecation is documented in the release notes.

See the [Deprecation Policy](../16-governance-and-process/08-deprecation-policy.md).

## Capability versioning

Capability identifiers are immutable. Once `input.touch` is registered, it
cannot change meaning. If a capability's semantics must change, a new
capability with a different identifier is created, and the old one is
deprecated.

## SDK and runtime versioning

The Platform API version is independent of:
- The PlayOS Runtime version.
- The PlayOS Shell version.
- The device profile version.

An application compiled against Platform API `0.1.0` SHOULD run on a
Runtime at version `0.2.0` without modification.

## Related

- [Deprecation Policy](../16-governance-and-process/08-deprecation-policy.md)
- [Compatibility Policy](../16-governance-and-process/07-compatibility-policy.md)
- [Long-Term Compatibility](../02-platform-principles/08-long-term-compatibility.md)
