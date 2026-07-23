---
rfc: 0017
title: Developer Experience and SDK Model
status: Draft
authors: []
created: 2026-07-22
---

# RFC-0017: Developer Experience and SDK Model

> **Status:** Draft

## Summary

This RFC defines the PlayOS developer experience: the SDK structure,
CLI tools, development packages, side-loading and deployment workflows,
debugging protocol, and the developer mode lifecycle.  It provides the
foundation for Part XV (Developer Guide) of the PlayOS Book and the
`playos-tools` repository.

## Motivation

A platform is only as good as its developer experience.  Without a
well-defined SDK and toolchain:

- Developers would struggle to set up a project, build, test, and
  deploy.
- Each developer would invent their own workflow.
- Debugging on console hardware would be inconsistent.
- The feedback loop (edit → build → deploy → test) would be slow.

A formal SDK model makes PlayOS approachable for indie developers and
efficient for professional studios.

## Guide-Level Explanation

The PlayOS SDK is a CMake-based C++17 toolchain that provides:

- **`playos-platform-api`** — headers and library you link against.
- **`playos-tools`** — CLI for packaging, deploying, and debugging.
- **Templates** — starter projects for Raylib, SDL, and plain C++.
- **Integration kits** — bridge your engine's input to PlayOS buttons.

You write your game in C++ (or any language with C FFI), build with
CMake, package with `playos pack`, deploy to a device with
`playos deploy`, and debug with `playos debug`.  The loop is:

```sh
cmake --build build
playos pack ./build -o MyGame.gpk
playos deploy MyGame.gpk --device rogue-ally
playos debug --device rogue-ally
```

## Reference-Level Explanation

### 1. SDK Structure

```
playos-sdk/
├── include/
│   └── playos/           # Platform API headers
├── lib/
│   └── libplayos.a       # Platform API library
├── cmake/
│   └── playos-config.cmake  # CMake package config
├── templates/
│   ├── raylib-starter/
│   ├── sdl-starter/
│   └── plain-cpp-starter/
├── tools/
│   └── playos             # CLI binary
└── docs/
    └── index.html         # API documentation
```

#### Installation

The SDK is distributed as:

| Format | Platform | Command |
|---|---|---|
| Tarball | Linux, macOS | `tar xf playos-sdk-1.0.0-linux-x86_64.tar.gz` |
| Zip | Windows | `Expand-Archive playos-sdk-1.0.0-windows-x86_64.zip` |
| APT repo | Debian/Ubuntu | `apt install playos-sdk` |
| Homebrew | macOS | `brew install playos-sdk` |
| Source | Any | `git clone && cmake --build` |

### 2. CLI Tools (`playos`)

The `playos` CLI is the developer's primary interface:

```sh
playos --help
# Commands:
#   new       Create a new project from a template
#   build     Build a project
#   pack      Package a built project into .gpk
#   deploy    Deploy a .gpk to a device
#   debug     Attach debugger to a running app
#   log       Stream logs from a device
#   profile   Profile an app (CPU, memory, GPU)
#   device    Manage connected devices
#   store     Interact with a marketplace
```

#### `playos new`

```sh
playos new my-game --template raylib
# Creates:
#   my-game/
#   ├── CMakeLists.txt
#   ├── src/main.cpp
#   ├── assets/
#   └── manifest.json
```

Templates include:
- **raylib-starter** — Raylib window + PlayOS input/lifecycle loop.
- **sdl-starter** — SDL window + PlayOS input bridging.
- **plain-cpp-starter** — Headless app (no rendering engine).

#### `playos pack`

```sh
playos pack ./build -o MyGame.gpk
# Options:
#   --sign-key path/to/key    Sign with Ed25519 key
#   --channel stable|beta     Set release channel
#   --arch x86_64|aarch64     Target architecture
```

Produces a `.gpk` package per RFC-0005.

#### `playos deploy`

```sh
playos deploy MyGame.gpk --device rogue-ally
# Options:
#   --device <id>           Target device (from `playos device list`)
#   --ip <addr>             Deploy over network (for remote devices)
#   --usb                   Deploy over USB
#   --dev-mode              Install as development package
```

Deployment transports:
- **USB** — ADB or PlayOS Debug Protocol over USB.
- **Network** — SSH or PlayOS Debug Protocol over TCP.
- **SD card** — Copy `.gpk` to removable storage; Shell auto-detects.

#### `playos debug`

```sh
playos debug --device rogue-ally
# Attaches debugger to running app. Supports:
#   - breakpoints
#   - stack trace
#   - variable inspection
#   - memory and register views
```

Uses GDB/LLDB remote protocol.  The Runtime exposes a debug stub on
the device in developer mode.

#### `playos log`

```sh
playos log --device rogue-ally --level info
# Streams application logs in real time:
# [2026-07-22 20:15:03.421] [INFO] [Game] Level 3 loaded
# [2026-07-22 20:15:03.450] [WARN] [Audio] Buffer underrun
```

### 3. Development Packages

Development packages are `.gpk` files with:

```json
{
  "devMode": true,
  "debugSymbols": true,
  "sourceMap": "src/"
}
```

Development packages:
- Do not require a valid signature (unsigned or self-signed).
- Include debug symbols (not stripped).
- Include source mapping for debugging.
- Are marked as "Development" in the Shell library with a distinct
  icon.
- Cannot be published to a store.

### 4. Side-Loading

Side-loading is installing a `.gpk` without going through a store:

| Method | Requires | Use Case |
|---|---|---|
| `playos deploy` | Developer mode on device | Active development |
| USB storage | Developer mode on device | Sharing builds |
| Network URL | Developer mode on device | CI/CD artifacts |
| QR code | Developer mode on device | Quick test on a device |
| Store install | No dev mode required | Distribution |

#### Side-loading flow (USB)

1. Developer copies `.gpk` to USB drive.
2. Inserts USB drive into PlayOS device.
3. Shell detects new package and shows notification.
4. Player opens notification → install confirmation.
5. Package is installed to the library.

### 5. Debugging Protocol

The PlayOS Debug Protocol is a TCP-based protocol for development tools:

```
playos-debug://<device>:<port>
```

#### Protocol features

| Feature | Protocol |
|---|---|
| Process attach/detach | GDB Remote Serial Protocol |
| Breakpoints | GDB RSP |
| Stack inspection | GDB RSP |
| Memory read/write | GDB RSP |
| Log streaming | Custom binary protocol |
| Performance counters | Custom binary protocol |
| Frame capture | Custom binary protocol |

#### Security

- Debug protocol is ONLY available in developer mode.
- Debug protocol binds to `127.0.0.1` only (not externally accessible).
- SSH tunneling is required for remote debugging.

### 6. Developer Mode Lifecycle

Developer mode is enabled in Settings → System → Developer Mode:

```
┌──────────────────────────────────────────┐
│  Developer Mode                          │
│                                          │
│  [●] Enabled                             │
│                                          │
│  When enabled:                           │
│  ✅ Side-load unsigned packages           │
│  ✅ Connect debugger (GDB/LLDB)           │
│  ✅ Access device shell (SSH/ADB)         │
│  ✅ View performance overlay              │
│  ✅ Stream logs in real time              │
│                                          │
│  ⚠ Developer mode reduces security.       │
│    Only enable if you are developing      │
│    applications.                          │
└──────────────────────────────────────────┘
```

#### Automatic disabling

Developer mode automatically disables after **72 hours** of
inactivity (no developer tools connected).  This prevents accidentally
leaving a device in an insecure state.

#### Developer mode and certification

Certified devices ship with developer mode disabled.  Enabling it
does not void certification, but it is not the expected consumer
state.

### 7. CI/CD Integration

The `playos` CLI is designed for CI/CD pipelines:

```yaml
# GitHub Actions example
- name: Build and package
  run: |
    cmake -B build
    cmake --build build
    playos pack ./build -o MyGame.gpk --channel beta

- name: Deploy to test device
  run: |
    playos deploy MyGame.gpk --device ci-device-01

- name: Run ACTS
  run: |
    playos test --device ci-device-01 --suite acts
```

### 8. Documentation and Learning Path

The developer guide (Part XV) provides:

| Guide | Audience |
|---|---|
| Installing the SDK | First-time setup |
| Your First PlayOS App | New developers |
| Raylib Starter Project | Game developers |
| Using Capabilities | All developers |
| Packaging Your Game | Pre-release |
| Publishing to a Store | Release |
| Debugging | All developers |
| Performance Guidelines | Optimization |
| Release Checklist | Pre-release |

## Drawbacks

- Requiring CMake adds a build dependency that some developers may
  not be familiar with (especially coming from Unity/Godot).
- The debug protocol being GDB-based ties the toolchain to GDB/LLDB;
  other debuggers would need custom adapters.
- 72-hour developer mode auto-disable may interrupt long development
  sessions (e.g., game jams).

## Rationale and Alternatives

- **IDE-based workflow (VS Code / Visual Studio plugin):** Would be
  more familiar to many developers but couples the platform to specific
  IDEs.  CLI-first with IDE plugins as optional provides broader
  compatibility.
- **`playos` CLI in Rust vs. Python vs. C++:** Rust is chosen for the
  CLI (as stated in the repository map) for fast startup, single-binary
  distribution, and safety.  Python would be simpler but slower and
  requires a Python runtime on the developer's machine.
- **Web-based debugger vs. GDB RSP:** A web debugger would be more
  accessible but requires a web server on the device.  GDB RSP is a
  well-established protocol supported by all major debuggers.
- **No developer mode; always allow side-loading:** Insecure for
  consumer devices.  Developer mode provides a clear security boundary.

## Prior Art

- **Android SDK** (`adb`, `android` CLI, Gradle): the closest analogue.
  `playos` CLI fills the same role as `adb` + `gradle` for Android
  development.
- **Nintendo Switch dev kit** (NDA, locked hardware, proprietary tools):
  PlayOS inverts this — open tools, consumer hardware, no NDA.
- **Steamworks SDK** (C++ API, CLI uploader, Steam client for testing):
  reference for the developer-to-platform workflow.
- **Playdate SDK** (Lua/C, simulator, USB deploy): small-scale console
  SDK that demonstrates the value of a simple, integrated toolchain.

## Unresolved Questions

- Should there be a simulator/emulator for development without physical
  hardware?  A PlayOS Simulator (running the Shell in a window on the
  developer's machine) would accelerate the edit-test cycle.
- How are multiple SDK versions managed side-by-side?  The SDK should
  support parallel installation of multiple MAJOR versions.
- Should `playos` CLI be available as a single static binary, or
  require a runtime (like Node.js or Python)?  Static binary is
  preferred.

## Future Possibilities

- **PlayOS Simulator:** A desktop application that runs the Shell and
  compositor in a window, emulating a PlayOS device for development
  without hardware.
- **IDE plugins:** VS Code, CLion, and Visual Studio extensions that
  wrap the `playos` CLI with GUI workflows.
- **Hot reload:** Reload assets and code without restarting the
  application, for faster iteration.
- **Cloud build service:** Upload source, get back a `.gpk` — no local
  toolchain needed.
- **PlayOS Dev Portal:** Web dashboard for managing apps, viewing
  analytics, and responding to player feedback.
