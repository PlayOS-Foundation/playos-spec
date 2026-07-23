# Platform Specification

The **Platform Specification** is the formal definition of what
constitutes a PlayOS platform. Implementations conform to the
specification — the specification does not conform to any
implementation.

## What the specification defines

| Artifact | Description |
|---|---|
| Platform API | C++ header files defining types, functions, and capabilities |
| Capability Registry | Canonical list of all capabilities and their semantics |
| Package Format | .gpk file structure, manifest schema, signing |
| Device Profile Schema | TOML format for hardware declarations |
| Cloud API | Protocol Buffer definitions for cloud services |
| Certification Tests | Conformance test suite |

## What the specification does NOT define

- Implementation details (how backends work internally).
- Runtime service internals (IPC protocol, process model).
- Shell UI design (visual style, layout).
- Build system or toolchain choices.
- Hardware requirements.

## Specification as source of truth

The specification is the **source of truth** for all PlayOS
repositories. If code in `playos-platform-api` contradicts the
specification, the specification wins. Update the specification
first, then change the code.

## Versioning the specification

The specification is versioned as a whole:

```toml
# /VERSION
[specification]
version = "0.4.0"
status = "draft"
```

Individual components (Platform API, package format, etc.) version
within the specification version. A specification version bump implies
that some component changed — see the changelog for details.

## Specification artifacts

| Artifact | Location |
|---|---|
| Platform API headers | `playos-platform-api/include/playos/` |
| Capability Registry | `schemas/capability-registry.schema.json` |
| Package Manifest Schema | `schemas/package-manifest.schema.json` |
| Device Profile Schema | `schemas/device-profile.schema.json` |
| Cloud API Protos | `playos-cloud-proto/` |
| Book | `book/src/` |
| RFCs | `rfcs/` |
| ADRs | `adr/` |

## Related

- [Layered Architecture](02-layered-architecture.md)
- [Capability Model](../05-capability-model/01-overview.md)
- [Platform API Overview](../06-platform-api/01-overview.md)
- [Governance](../16-governance-and-process/01-overview.md)
