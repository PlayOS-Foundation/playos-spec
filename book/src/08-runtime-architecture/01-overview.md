# Runtime Architecture Overview

The **PlayOS Runtime** is the software layer that sits between the
Platform API and the operating system. It provides the full "console"
experience — game switching, overlay, cloud services, package
management, and system UI.

## Architecture

```
┌──────────────────────────┐
│    PlayOS Applications   │  ← .gpk packages
├──────────────────────────┤
│     Platform API         │  ← public C++ API
├──────────────────────────┤
│  Platform Backends       │  ← per-OS implementations
├──────────────────────────┤
│  Runtime Services        │  ← overlay, IPC, lifecycle
├──────────────────────────┤
│  Operating System        │  ← Linux, Windows, Android
└──────────────────────────┘
```

On Runtime Devices, the Runtime and the Shell work together to provide
the full experience. On SDK Targets, applications link directly to the
Platform API without the Runtime.

## What the Runtime provides

| Service | Description |
|---|---|
| Application lifecycle | Launch, switch, terminate applications |
| Overlay | System UI drawn over running applications |
| Input routing | Routes input to the active application or the shell |
| Package management | Install, verify, update, and remove .gpk packages |
| Cloud sync | Synchronize saves, achievements, leaderboards |
| Suspend / resume | Save application state and restore |
| Security sandbox | Isolate applications from each other and the system |
| System UI | Home screen, quick settings, notifications |

## Design principles

1. **The Platform API is the contract** — everything above it is
   portable. Everything below it is target-specific.
2. **Isolation** — applications are isolated from each other and from
   the system.
3. **Background services** — the Runtime runs as a collection of
   cooperating processes communicating over IPC.
4. **Engine agnostic** — the Runtime does not depend on Raylib or any
   specific engine.
5. **Self-hostable** — all services that touch the cloud can be
   self-hosted.

## Chapter index

| Chapter | Description |
|---|---|
| [Package Execution](07-package-execution.md) | Running .gpk applications |
| [Runtime Services](08-runtime-services.md) | Services that make up the Runtime |
| [Runtime IPC](09-runtime-ipc.md) | Inter-process communication |
| [Input Routing](10-input-routing.md) | How input reaches applications |
| [Game Switching](11-game-switching.md) | Switching between running applications |
| [Overlay Integration](12-overlay-integration.md) | System overlay system |
| [Audio Startup](13-audio-startup.md) | Audio device enumeration |
| [Network Startup](14-network-startup.md) | Network interface enumeration |
| [Updates](15-updates.md) | System and application updates |
| [Suspend and Resume](16-suspend-and-resume.md) | Application state preservation |
| [Security Model](17-security-model.md) | Sandboxing and permissions |
| [Observability](18-observability.md) | Logging, metrics, diagnostics |

## Related

- [Target Model](../04-target-model/01-overview.md)
- [Package Format](../10-package-format/01-overview.md)
- [Shell and UX](../09-shell-and-ux/01-overview.md)
