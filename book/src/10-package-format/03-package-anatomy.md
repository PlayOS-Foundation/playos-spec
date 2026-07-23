# Package Anatomy

A `.gpk` package is a ZIP archive with a defined internal directory
structure. Every file at the root has a specific purpose; additional files
are ignored by the runtime.

## Directory layout

```text
MyGame.gpk
├── manifest.json       # REQUIRED. Package identity, metadata, and declarations.
├── signature           # OPTIONAL. Detached signature (required for store).
├── bin/                # REQUIRED. Entry executable(s).
│   ├── x86_64/mygame   # Linux x86_64 executable
│   └── aarch64/mygame  # Linux aarch64 executable
├── assets/             # OPTIONAL. Game assets (any structure).
├── icon.png            # REQUIRED. Square icon (recommended 512×512).
├── banner.png          # OPTIONAL. Wide banner for store display (recommended 1920×640).
└── screenshots/        # OPTIONAL. Store screenshots (PNG or JPEG).
```

## Required files

| File | Purpose |
|---|---|
| `manifest.json` | Identity, version, entry point, permissions, capabilities |
| `bin/` | At least one executable matching the target architecture |
| `icon.png` | Square icon displayed in the library and store |

## Optional files

| File | Purpose |
|---|---|
| `signature` | Detached cryptographic signature over the archive contents |
| `banner.png` | Wide banner image for store listings |
| `screenshots/` | Screenshots for store listings (up to 10) |
| `assets/` | Game assets — any directory structure, the runtime does not inspect |

## Archive format

- **Container:** ZIP (Deflate compression).
- **Path separator:** Forward slash (`/`).
- **Encoding:** UTF-8 for all text files.
- **Maximum size:** No hard limit; the marketplace may impose practical
  limits (recommended: under 8 GB per package for store distribution).

## Entry point

The `manifest.json` `entry` field points to the executable within `bin/`:

```json
{"entry": "bin/mygame"}
```

The runtime appends the architecture subdirectory when resolving:
`bin/x86_64/mygame` on x86_64 devices, `bin/aarch64/mygame` on ARM64.

## Related

- [Manifest](04-manifest.md)
- [Executables](07-executables.md)
- [Architectures and ABIs](08-architectures-and-abis.md)
- RFC-0005 (Package Format)
