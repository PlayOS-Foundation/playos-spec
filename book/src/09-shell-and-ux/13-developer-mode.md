# Developer Mode

**Developer Mode** enables features needed for application development
and testing on Runtime Devices.

## Enabling developer mode

Developer mode is enabled in Settings → System → Developer Mode:

```
┌──────────────────────────────────────────┐
│  ← Developer Mode                        │
├──────────────────────────────────────────┤
│                                          │
│  Developer Mode               [●]        │
│                                          │
│  When enabled:                           │
│  • Side-load unsigned packages           │
│  • Access developer tools                │
│  • View performance overlay              │
│  • Connect debugger                      │
│  • Access device shell (SSH/ADB)         │
│                                          │
│  ⚠ Developer mode reduces security.      │
│    Only enable if you are developing     │
│    applications.                         │
└──────────────────────────────────────────┘
```

## Side-loading

In developer mode, unsigned packages can be installed:

- From USB storage.
- From network (HTTP URL or `playos-cli deploy`).
- From the `playos-tools` CLI over ADB/SSH.

Signed packages from the marketplace can always be installed,
regardless of developer mode.

## Developer tools

Developer mode enables:

| Tool | Description |
|---|---|
| Performance HUD | FPS, frame time, CPU/GPU usage overlay |
| Frame debugger | Capture and inspect frames |
| Memory profiler | Heap allocation tracking |
| GPU profiler | Draw call and shader analysis |
| Log viewer | Real-time application log stream |
| Crash log viewer | Inspect local crash dumps |

## Remote access

Developer mode enables remote access for debugging:

- **SSH** — shell access to the device.
- **ADB** — Android Debug Bridge (for Android-based devices).
- **PlayOS Debug Protocol** — attach the PlayOS debugger from
  `playos-tools`.

## Security implications

Developer mode relaxes security:

- Unsigned packages can run.
- Remote shell access is enabled.
- Applications can access more system information (logs, crash dumps).

Developer mode should never be enabled on a production device used by
end users.

## Automated disabling

Developer mode automatically disables after 72 hours of inactivity
(no developer tools connected). This prevents accidentally leaving
developer mode enabled on a device that returns to normal use.

## Related

- [Development Packages](../10-package-format/15-development-packages.md)
- [Tools](../03-platform-architecture/10-tools.md)
- [Security Model](../08-runtime-architecture/17-security-model.md)
- [Observability](../08-runtime-architecture/18-observability.md)
