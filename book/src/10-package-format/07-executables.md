# Executables

The `bin/` directory contains the application entry point — the executable
the runtime launches when the player starts the application.

## Entry point

The `manifest.json` `entry` field specifies the path to the executable:

```json
{"entry": "bin/mygame"}
```

The runtime resolves the full path by inserting the architecture
subdirectory:

```text
manifest "entry": "bin/mygame"
   → bin/x86_64/mygame    on x86_64 devices
   → bin/aarch64/mygame   on ARM64 devices
```

## Supported formats

| Platform | Format |
|---|---|
| Linux (reference runtime) | ELF executable, dynamically linked |
| Windows (SDK Target) | PE executable (`.exe`) |
| macOS (SDK Target) | Mach-O executable or `.app` bundle |
| Android (SDK Target) | Native shared library (`.so`) loaded by a wrapper |
| PS Vita (SDK Target) | ELF executable (`.velf`) |

## Execution model

1. The runtime extracts the package to a private directory.
2. Resolves the entry point for the device architecture.
3. Sets the working directory to the extracted package root.
4. Launches the executable as a child process (Runtime Device) or invokes
   it directly (SDK Target).
5. Monitors the process for lifecycle events (exit, crash, suspend).

## Multiple executables

A package may include multiple executables (launcher, configuration tool,
server). Only the one referenced by `entry` is launched by default.
Additional executables are not directly accessible to the player through
the standard shell UI.

## Requirements

- The entry executable MUST be self-contained — the runtime does not
  install dependencies or run scripts before launch.
- Executables MUST NOT require root privileges.
- On Linux, the executable SHOULD link against a known set of libraries
  (glibc, libstdc++, OpenGL/Vulkan, PulseAudio/PipeWire). Additional
  libraries may be bundled in `lib/`.

## Related

- [Architectures and ABIs](08-architectures-and-abis.md)
- [Package Execution](../08-runtime-architecture/07-package-execution.md)
- [Application Lifecycle](../08-runtime-architecture/06-application-lifecycle.md)
