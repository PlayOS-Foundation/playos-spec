# Runtime Services

The PlayOS Runtime is composed of independent **services** that
communicate over IPC. Each service has a single responsibility and runs
in its own process (or thread) for isolation.

## Service list

| Service | Responsibility | Capability |
|---|---|---|
| **Application Manager** | Launch, switch, terminate apps | `system.app-management` |
| **Overlay Service** | Draw system UI over running apps | `system.overlay` |
| **Input Router** | Route input to active app or shell | `input.basic` (required) |
| **Package Installer** | Install, verify, update, remove .gpk | `system.package-install` |
| **Cloud Sync** | Sync saves, achievements, leaderboards | `cloud.saves` and others |
| **Audio Service** | Enumerate audio devices, volume control | `audio.core` |
| **Network Manager** | Enumerate interfaces, manage connections | `network.info` |
| **Security Monitor** | Enforce sandbox, permissions | `system.security` |
| **Lifecycle Manager** | Suspend, resume, hibernate apps | `lifecycle.basic` |
| **Update Service** | Check for and apply system/app updates | `system.updates` |
| **Logging Service** | Collect and forward diagnostics | `system.logging` |
| **Shell UI** | Home screen, quick settings, launcher | `system.shell` |

## Service architecture

```
┌─────────────────────────────────────────────┐
│                 Runtime IPC Bus             │
├────┬──────┬──────┬──────┬──────┬────────────┤
│ AM │ Over │ Input│ Pkg  │Cloud │ Lifecycle  │
│    │ lay  │Route │Inst. │ Sync │  Manager   │
└────┴──────┴──────┴──────┴──────┴────────────┘
```

## Start-up order

1. Security Monitor — establishes sandbox.
2. Logging Service — captures all subsequent output.
3. Input Router — ready for input immediately.
4. Audio and Network — enumerate devices.
5. Package Installer — scan for installed applications.
6. Application Manager — ready to launch.
7. Shell UI — displayed once everything is ready.
8. Cloud Sync and Update Service — background, non-blocking.

## Service failure recovery

- If a service crashes, the Security Monitor restarts it.
- Application state is preserved across service restarts.
- The Shell remains responsive even if Cloud Sync is unavailable.

## Related

- [Runtime IPC](09-runtime-ipc.md)
- [Overlay Integration](12-overlay-integration.md)
- [Security Model](17-security-model.md)
- [Shell and UX](../09-shell-and-ux/01-overview.md)
