# Logging and Diagnostics API

The Logging and Diagnostics API provides structured logging, error reporting,
and performance counters for applications. It is available on all conformant
targets (no capability gate).

## Purpose

- **Structured logging** — emit log messages with severity levels.
- **Error reporting** — report non-fatal errors to the platform.
- **Performance counters** — measure frame time and other metrics.
- **Crash reporting integration** — logs feed into the crash reporting
  pipeline on Runtime Devices.

## API concepts

```cpp
namespace PlayOS {
namespace Log {

enum class Level {
    Debug,
    Info,
    Warning,
    Error,
    Fatal
};

// Emit a log message at the given severity.
void Write(Level level, const char* message);

// Set the minimum severity that is recorded.
void SetMinimumLevel(Level level);

// Measure frame timing.
void BeginFrame();   // call at start of frame
void EndFrame();     // call at end of frame
float LastFrameMs(); // duration of the last completed frame in milliseconds

} // namespace Log
} // namespace PlayOS
```

## Log output

On Runtime Devices, log output is captured by the runtime's logging daemon
and forwarded to:
- A local ring buffer (viewable in Developer Mode).
- The cloud analytics pipeline (if telemetry is enabled).
- The crash reporter (as context for crash dumps).

On SDK Targets, log output goes to `stderr` by default.

## Integration with crash reporting

When an application crashes, the most recent log messages (from the ring
buffer) are attached to the crash report. Applications SHOULD use
`Log::Write` for diagnostic information that helps debug crashes.

## Requirements

- `Write` MUST be safe to call from any thread.
- `Write` MUST NOT allocate memory (it writes to a pre-allocated ring
  buffer).
- `LastFrameMs` MUST return the wall-clock duration between the most
  recent `BeginFrame` / `EndFrame` pair in milliseconds.
- Logging MUST be available without a capability query.

## Related

- [Crash Reporting](../11-cloud-and-marketplace/15-crash-reporting.md)
- [Developer Mode](../09-shell-and-ux/13-developer-mode.md)
- [Observability](../08-runtime-architecture/18-observability.md)
