# Architectures and ABIs

A single `.gpk` package may contain executables for multiple CPU
architectures. The runtime selects the correct one at launch time.

## Architecture subdirectories

Executables are organized under `bin/<arch>/`:

```text
bin/
├── x86_64/mygame       # 64-bit x86 (Linux, Windows, macOS)
├── aarch64/mygame      # 64-bit ARM (Linux, Android)
├── armv7l/mygame       # 32-bit ARM (older devices)
└── mipsel/mygame       # MIPS little-endian (PS Vita homebrew target)
```

## Architecture identifiers

| Identifier | Description | Typical devices |
|---|---|---|
| `x86_64` | 64-bit x86 | Desktop PC, ROG Ally, Steam Deck |
| `aarch64` | 64-bit ARM | Orange Pi, Android devices, Apple Silicon |
| `armv7l` | 32-bit ARM | Raspberry Pi, older Android |
| `mipsel` | MIPS little-endian | PS Vita (homebrew target) |
| `i686` | 32-bit x86 | Older PCs |

## Declaration

The manifest declares supported architectures:

```json
{
  "architecture": ["x86_64", "aarch64"]
}
```

If `architecture` is omitted, the runtime assumes `["x86_64"]`.

## Selection

At launch time, the runtime:

1. Reads the device's native architecture.
2. Looks for `bin/<native-arch>/<entry>`.
3. If found, launches it.
4. If not found, the installation fails with an architecture-mismatch
   error.

The runtime does not emulate foreign architectures. A package without
a matching executable cannot run.

## ABI compatibility

The ABI (Application Binary Interface) is defined by the target platform's
standard library and system calls. The Platform API abstracts OS-specific
details, so applications using the Platform API exclusively are portable
across OSes on the same architecture.

## Related

- [Executables](07-executables.md)
- [SDK Targets](../04-target-model/03-sdk-targets.md)
- [Package Validation](16-package-validation.md)
