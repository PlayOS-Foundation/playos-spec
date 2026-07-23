# Specification Status

The PlayOS Platform Specification is currently in **Draft** status
(version 0.4.0).

## What "Draft" means

- The specification is under active development.
- Breaking changes may occur between versions.
- APIs, schemas, and formats are not yet stabilized.
- RFCs are the primary mechanism for proposing changes.
- Implementations are prototyping and validating the specification.

## Stability timeline

| Version | Status | Stabilization |
|---|---|---|
| 0.x | Draft | Active development, breaking changes allowed |
| 1.0-alpha | Alpha | Feature-complete, no new APIs |
| 1.0-beta | Beta | Bug fixes only, API freeze |
| 1.0 | Stable | First stable release, long-term support |
| 1.x+ | Stable | [Semantic versioning](../06-platform-api/25-api-versioning.md) applies |

## What is stable now

The core concepts are stable:

- **Capability model** — the concept of querying capabilities at
  runtime is fundamental and will not change.
- **Device profiles** — the TOML format is unlikely to change
  significantly.
- **Package format** — the .gpk structure is stabilizing.
- **Platform principles** — the console-first, capability-based,
  self-hostable philosophy.

Specific API surfaces, capability names, and schema fields may change
during the draft phase.

## How to track changes

- **GitHub releases** — tagged versions of `playos-spec`.
- **Changelog** — `CHANGELOG.md` in the repository root.
- **RFCs** — major changes proposed through `rfcs/`.
- **ADRs** — architectural decisions recorded in `adr/`.

## Providing feedback

- Open an issue on the
  [playos-spec repository](https://github.com/PlayOS-Foundation/playos-spec).
- Comment on open RFCs.
- Join the discussion in
  [PlayOS Discussions](https://github.com/PlayOS-Foundation/playos-spec/discussions).

## Related

- [Normative Language](04-normative-language.md)
- [API Versioning](../06-platform-api/25-api-versioning.md)
- [Governance](../16-governance-and-process/01-overview.md)
