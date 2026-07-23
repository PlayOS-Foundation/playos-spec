# Package Validation

Package validation checks that a `.gpk` package conforms to the schema
and platform rules before installation or store upload.

## Validation checks

| Check | Description | Severity |
|---|---|---|
| **Archive integrity** | The `.gpk` is a valid ZIP archive | Error |
| **Manifest presence** | `manifest.json` exists at the package root | Error |
| **Schema conformance** | `manifest.json` validates against `package-manifest.schema.json` | Error |
| **Required files** | `icon.png` and at least one executable in `bin/` exist | Error |
| **Entry point** | The `entry` path exists for at least one architecture | Error |
| **Permission validity** | All declared permissions exist in the permission registry | Error |
| **Capability validity** | All declared capabilities exist in the capability registry | Warning |
| **Signature** | Signature is present and valid (required for store) | Error (store) / Warning (dev) |
| **Icon dimensions** | `icon.png` is square and at least 256×256 | Warning |
| **Banner dimensions** | `banner.png` (if present) has a 3:1 aspect ratio | Warning |
| **File path encoding** | All paths are valid UTF-8 and use forward slashes | Error |
| **Total size** | Package is under the marketplace size limit | Warning |

## Tooling

Validation is performed by `playos-tools`:

```sh
# Validate a package for store upload.
playos-tools validate --strict MyGame.gpk

# Validate a development package (warnings only).
playos-tools validate MyGame.gpk
```

The marketplace runs `--strict` validation before accepting an upload.
The runtime runs validation before installation, blocking on errors.

## Requirements

- Schema validation MUST use the published
  `package-manifest.schema.json`.
- Archive integrity MUST be checked before extraction.
- Error-level checks MUST block installation and store upload.
- Warning-level checks MUST be reported but not block installation.

## Related

- `schemas/package-manifest.schema.json`
- [Manifest](04-manifest.md)
- [Signing](10-signing.md)
- [CLI Reference](../99-appendices/08-cli-reference.md)
- RFC-0005 (Package Format)
