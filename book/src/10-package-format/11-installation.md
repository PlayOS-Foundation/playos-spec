# Installation

Installation is the process of extracting a `.gpk` package to the
device, recording it in the local package database, and obtaining player
consent for declared permissions.

## Process

1. **Verify** — the runtime verifies the package signature (if present)
   and validates the manifest against the schema.
2. **Present permissions** — the runtime displays the declared permissions
   to the player. The player may grant or deny each.
3. **Extract** — the runtime extracts the package to a private directory
   under `<data-root>/packages/<id>/`.
4. **Record** — the runtime records the package identity, version,
   permissions, and install path in the local package database.
5. **Register with shell** — the shell adds the application to the
   library and game switcher.

## Install location

```text
<data-root>/packages/com.studio.mygame/
├── manifest.json
├── bin/...
├── assets/...
├── icon.png
└── ... (extracted package contents)
```

The exact `<data-root>` is platform-specific: `/var/lib/playos/` on
Linux, `%APPDATA%/PlayOS/` on Windows.

## Post-install

After installation:
- The application appears in the library.
- The player can launch it immediately.
- Permissions are applied as granted during install.
- Save data is stored separately (see [Storage API](../06-platform-api/10-storage-api.md))
  and survives updates and reinstalls.

## Installation failures

If installation fails at any step:
- Partially extracted files MUST be cleaned up.
- No database entry is created.
- An error is reported to the shell (visible in Downloads or the
  notification center).

## Requirements

- Installation MUST be atomic from the player's perspective — either the
  application is fully installed or not installed at all.
- The runtime MUST NOT overwrite an existing installation of the same
  package without explicit player confirmation (update flow).
- Permission consent MUST be obtained before extraction.

## Related

- [Uninstallation](12-uninstallation.md)
- [Updates and Patches](13-updates-and-patches.md)
- [Package Execution](../08-runtime-architecture/07-package-execution.md)
