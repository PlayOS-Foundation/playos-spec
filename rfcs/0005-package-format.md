---
rfc: 0005
title: Package Format
status: Accepted
authors: []
created: 2026-01-01
---

# RFC-0005: Package Format

> **Status:** Accepted

## Summary

PlayOS applications are distributed as `.gpk` packages — self-contained,
signed archives containing an executable, assets, a manifest, and metadata.
This RFC defines the package anatomy, the manifest schema, permissions,
signing requirements, and the install/update/uninstall model. Schema:
[`schemas/package-manifest.schema.json`](https://github.com/PlayOS-Foundation/playos-spec/blob/main/schemas/package-manifest.schema.json).

## Motivation

Without a standard package format, every PlayOS application would require a
custom installer. A unified format lets the runtime install, launch, track,
update, and uninstall applications consistently. It enables the marketplace
(discovery + distribution) and certification (validation against the schema).

## Guide-Level Explanation

A `.gpk` package is a ZIP archive containing:

```text
MyGame.gpk
├── manifest.json       # package identity and metadata
├── bin/                # entry executable (per-arch variants)
├── assets/             # game assets
├── icon.png            # square icon for the library
├── banner.png          # optional wide banner for the store
├── screenshots/        # optional
└── signature           # detached signature over the archive
```

A minimal manifest:

```json
{"schemaVersion":"0.1.0","id":"com.studio.mygame","name":"My Game",
 "version":"1.0.0","entry":"bin/mygame",
 "permissions":["storage.local"],"capabilities":["input.basic","lifecycle.basic"]}
```

## Reference-Level Explanation

### Identity
`id` is a reverse-DNS identifier, globally unique and immutable after first
publication. `version` is semantic (MAJOR.MINOR.PATCH).

### Permissions
Declared in the manifest; surfaced to the player at install time where user
consent is required. The permission registry defines available permissions
and risk levels.

### Signing
Packages for store distribution MUST be signed; dev/local packages MAY be
unsigned. The runtime MUST verify the signature before installation when
present.

### Lifecycle
- **Install:** extract to a private directory; record in the local package
database.
- **Launch:** resolve entry point and execute per the package-execution
model.
- **Update:** replace the extracted bundle; preserve save data (lives
externally).
- **Uninstall:** remove bundle + DB record; save data removal is optional.

### Multi-arch
`manifest.json` may specify `"architecture": ["x86_64", "aarch64"]`.
Arch-specific executables live under `bin/<arch>/`.

## Drawbacks

- A custom format means PlayOS cannot reuse existing package managers
  (Flatpak, AppImage). Accepted: PlayOS needs control over permissions,
  signing, and the console-style lifecycle.
- Tooling must exist (`playos-tools`).

## Rationale and Alternatives

- **Flatpak / AppImage:** Rejected — carry a desktop-oriented runtime and
  permission model that does not map to PlayOS capabilities/profiles.
- **Plain zip, no manifest:** Rejected — no identity, permissions, or
  version tracking.
- **Custom binary container:** Deferred — ZIP is sufficient and easier to
  inspect; a more efficient container can be added later.

## Unresolved Questions

- Delta updates (binary patches) are deferred.
- On-disk layout of the local package database is a runtime implementation
detail.
- Whether the signature covers the whole archive or a manifest of hashes is
  decided in the signature appendix.
