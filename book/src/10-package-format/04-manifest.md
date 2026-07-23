# Manifest

The `manifest.json` file at the root of every `.gpk` package is the
single source of truth for package identity, metadata, and declarations.

## Schema

The manifest follows [`package-manifest.schema.json`](../../schemas/package-manifest.schema.json).
A minimal manifest:

```json
{
  "schemaVersion": "0.1.0",
  "id": "com.studio.mygame",
  "name": "My Game",
  "version": "1.0.0",
  "entry": "bin/mygame",
  "permissions": ["storage.local"],
  "capabilities": ["input.basic", "lifecycle.basic"]
}
```

## Fields

| Field | Required | Description |
|---|---|---|
| `schemaVersion` | Yes | Manifest schema version (`"0.1.0"`) |
| `id` | Yes | Globally unique reverse-DNS identifier, immutable after first publication |
| `name` | Yes | Human-readable title |
| `version` | Yes | Semantic version (`MAJOR.MINOR.PATCH`) |
| `entry` | Yes | Path to the entry executable relative to package root |
| `permissions` | Yes | List of permission identifiers the application requires |
| `capabilities` | No | List of capability identifiers the application uses (informational) |
| `architecture` | No | List of supported architectures (defaults to `["x86_64"]`) |
| `description` | No | Short description for store display |
| `author` | No | Developer or studio name |
| `minApiVersion` | No | Minimum Platform API version required |
| `minRuntimeVersion` | No | Minimum PlayOS Runtime version required (runtime-only apps) |

## Identity (`id`)

The `id` is a reverse-DNS identifier, globally unique, and immutable after
first publication:

```
com.studio-name.game-name
org.indie-dev.my-app
```

Changing the `id` creates a new package identity. Updates to the same
application keep the same `id`.

## Version (`version`)

Semantic versioning: `MAJOR.MINOR.PATCH`.

- **MAJOR** — incompatible changes (save data format, removed content).
- **MINOR** — new features, backward-compatible.
- **PATCH** — bug fixes, no new features.

The runtime uses the version to determine update eligibility (newer version
→ update available).

## Permissions

The `permissions` array declares every permission the application may use.
Permissions are granted at install time by the player. See
[Permissions](09-permissions.md).

## Capabilities

The `capabilities` array is informational — it declares which capabilities
the application expects. The runtime does not enforce this list at install
time (capabilities are discovered at runtime), but the marketplace may use
it to filter listings by device compatibility.

## Related

- `schemas/package-manifest.schema.json`
- [Permissions](09-permissions.md)
- [Package Metadata](05-package-metadata.md)
- RFC-0005 (Package Format)
