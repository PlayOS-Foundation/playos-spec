# Overview

PlayOS applications are distributed as `.gpk` packages — self-contained,
signed archives that contain everything needed to install, launch, and
manage an application on a PlayOS device.

## What a .gpk package is

A `.gpk` is a ZIP archive with a defined internal structure:

```text
MyGame.gpk
├── manifest.json       # identity, metadata, permissions, capabilities
├── bin/                # entry executable (per-architecture variants)
├── assets/             # game assets
├── icon.png            # square icon for the library
├── banner.png          # optional wide banner for the store
├── screenshots/        # optional store screenshots
└── signature           # detached signature over the archive
```

## Why a custom format

PlayOS uses its own format rather than Flatpak, AppImage, or platform-native
packages because:

1. **Console-style lifecycle** — install, launch, suspend, update, uninstall
   are managed by the runtime, not by a desktop package manager.
2. **Permissions and capabilities** — declared in the manifest and enforced
   by the runtime, not by OS-level sandboxing.
3. **Signing and trust** — the signature model is designed for a console
   marketplace, not a desktop repository.
4. **Portability** — a `.gpk` must work on every PlayOS Runtime Device
   regardless of the underlying OS.

## Where packages fit

```text
Developer builds .gpk → signs → uploads to marketplace
   → player installs from store → runtime verifies, extracts, launches
   → runtime manages updates, uninstalls
```

The package format is the contract between the developer and the runtime.
The marketplace stores and distributes packages but does not define their
internal structure.

## Related

- RFC-0005 (Package Format)
- `schemas/package-manifest.schema.json`
- [Package Execution](../08-runtime-architecture/07-package-execution.md)
