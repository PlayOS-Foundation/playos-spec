# Observability

The Runtime provides **observability** — structured logging, metrics,
and diagnostics — for debugging, performance analysis, and support.

## Logging

The Logging Service collects logs from all Runtime services and
applications. Applications emit logs through the Platform API:

```cpp
Log::Info("Game", "Level loaded: %s", levelName);
Log::Warning("Audio", "Buffer underrun on channel %d", channel);
Log::Error("Network", "Connection timeout to %s", host);
```

### Log levels

| Level | Use case |
|---|---|
| `Trace` | Detailed diagnostic information (dev mode only) |
| `Debug` | Debugging information (dev mode only) |
| `Info` | General operational messages |
| `Warning` | Recoverable issues |
| `Error` | Errors that affect functionality |
| `Fatal` | Errors that prevent the application from continuing |

### Log routing

Logs are routed to:

1. **Local file** — `/var/log/playos/` on the device.
2. **Developer console** — when connected via ADB or SSH.
3. **Cloud** — when the user opts in to telemetry.

## Metrics

The Runtime exposes metrics for monitoring:

| Metric | Description |
|---|---|
| Frame time | Time between frames (ms). Target: ≤16.6 ms for 60 FPS. |
| Frame drops | Frames that missed the vsync deadline. |
| Input latency | Time from button press to application receiving the event. |
| Memory usage | RSS, heap, GPU memory. |
| CPU usage | Per-thread CPU time. |
| Disk I/O | Read/write throughput. |
| Network I/O | Bytes sent/received. |

Metrics are accessed through the Platform API:

```cpp
auto frameMetrics = Diagnostics::GetFrameMetrics();
float avgFrameTime = frameMetrics.averageFrameTimeMs;
int frameDrops = frameMetrics.frameDropsLastSecond;
```

## Developer overlay

In development mode, the overlay includes a performance HUD:

```
FPS: 59.8   Frame: 16.2ms   CPU: 34%   GPU: 62%   RAM: 1.2GB
```

This is never shown in production (certified) builds.

## Crash reporting

When an application crashes:

1. The Security Monitor detects the crash.
2. A minidump is captured (application process only).
3. The crash report is stored locally.
4. If the user opts in, the report is uploaded to the cloud.

## Privacy

- Metrics and crash reports are **opt-in**.
- No personally identifiable information is collected.
- Frame-level metrics are never sent to the cloud (too granular).
- The user can disable telemetry at any time.

## Related

- [Logging and Diagnostics API](../06-platform-api/22-logging-and-diagnostics-api.md)
- [Runtime Services](08-runtime-services.md)
- [Developer Guide](../15-developer-guide/01-overview.md)
