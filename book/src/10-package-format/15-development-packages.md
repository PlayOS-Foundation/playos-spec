# Development Packages

Development packages are `.gpk` packages created during development —
unsigned, side-loaded, and intended for testing, not distribution.

## Characteristics

| Property | Development | Store |
|---|---|---|
| **Signing** | May be unsigned | Must be signed |
| **Installation** | Side-loaded via USB, network, or local path | Downloaded from marketplace |
| **Validation** | Loose (warnings, not errors) | Strict (must pass schema validation) |
| **Library display** | Marked "Dev" or "Unverified" | Normal display |

## Side-loading

Development packages are installed outside the marketplace:

1. Enable Developer Mode in Settings.
2. Transfer the `.gpk` to the device (USB, network share, `scp`).
3. Open the package from the file browser or use the CLI:
   ```sh
   playos-cli install ./MyGame.gpk
   ```
4. The runtime installs the package without signature verification.

## Developer Mode

Developer Mode enables:

- Side-loading unsigned packages.
- Access to the local package database and logs.
- Running packages directly from the command line.
- Connecting debuggers and profilers.
- Accessing the developer shell overlay.

Developer Mode is disabled by default and must be explicitly enabled in
Settings → System → Developer Mode.

## Requirements

- Development packages MUST be visually distinct in the library (icon
  badge or "Dev" label).
- Developer Mode MUST be off by default.
- The runtime MUST NOT allow development packages to masquerade as store
  packages.

## Related

- [Developer Mode](../09-shell-and-ux/13-developer-mode.md)
- [Package Validation](16-package-validation.md)
- [CLI Reference](../99-appendices/08-cli-reference.md)
