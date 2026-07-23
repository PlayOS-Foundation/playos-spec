# Long-Term Compatibility

PlayOS guarantees **long-term compatibility**: applications compiled against
one version of the Platform API continue to work on future versions without
modification or recompilation.

## The guarantee

> Within a MAJOR version, an application binary compiled against version
> `0.N` runs on version `0.N+K` without recompilation.

Breaking changes are rare, versioned explicitly (MAJOR increment), and
preceded by a deprecation period of at least one MINOR version.

## What is covered

- **API surface** — function signatures, enum values, struct layouts,
  capability identifiers.
- **Behavior** — documented semantics do not change within a MAJOR
  version.
- **Package format** — `.gpk` packages remain installable across
  Runtime versions.

## What is NOT covered

- **Undocumented behavior** — if it is not in the specification, it may
  change.
- **Experimental APIs** — marked as such in the API index.
- **Engine compatibility** — the engine is the developer's choice and is
  not covered by the Platform API compatibility guarantee.

## Deprecation process

1. **Announce** — a symbol is marked deprecated in a MINOR release.
2. **Warn** — the compiler emits a deprecation warning for one full
   MINOR cycle.
3. **Remove** — the symbol is removed in the next MAJOR release.

See [Deprecation Policy](../16-governance-and-process/08-deprecation-policy.md).

## Compatibility window

The Platform API guarantees backward compatibility for at least **two
years** after the next MAJOR version is released. This gives developers
time to update without pressure.

## Why this matters

Console platforms traditionally guarantee compatibility for the life of
the hardware generation. PlayOS extends this to the platform API itself:
a game released today should run on PlayOS devices five years from now.

## Related

- [API Versioning](../06-platform-api/25-api-versioning.md)
- [Deprecation Policy](../16-governance-and-process/08-deprecation-policy.md)
- [Compatibility Policy](../16-governance-and-process/07-compatibility-policy.md)
