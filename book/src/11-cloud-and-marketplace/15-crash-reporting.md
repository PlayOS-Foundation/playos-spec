# Crash Reporting

The **Crash Reporting** service collects crash dumps from devices to
help developers diagnose and fix bugs.

## How it works

1. An application crashes (segfault, unhandled exception, etc.).
2. The Security Monitor detects the crash.
3. A minidump is captured:
   - Stack trace for all threads.
   - Register state at crash point.
   - Loaded module list (with versions).
   - Application log buffer (last 1 MB).
4. The minidump is stored locally in `/var/log/playos/crashes/`.
5. If the player has opted in, the minidump is uploaded to the cloud.

## What is included

| Data | Included? |
|---|---|
| Stack trace | ✅ |
| Register state | ✅ |
| Loaded modules | ✅ |
| Application logs | ✅ (last 1 MB) |
| System info (OS, device, version) | ✅ |
| Application state / save data | ❌ |
| Screen capture | ❌ |
| Input history | ❌ |
| PII | ❌ |

## Developer access

Crash reports are available in the Developer Portal, grouped by
crash signature (stack trace hash). The dashboard shows:

- Crash rate (crashes per session).
- Top crash signatures.
- Affected versions.
- Device/OS breakdown.

## Symbolication

Developers upload debug symbols (`.debug` files or PDBs) through the
Developer Portal. The Crash Reporting service uses these to symbolicate
stack traces, turning raw addresses into function names and line
numbers.

## Player opt-out

Players can disable crash reporting in Settings → Privacy → Crash
Reports. When disabled, minidumps are stored locally but never
uploaded. The local crash log can still be shared manually with
support.

## Self-hosting

The Crash Reporting service is Sentry-compatible. Self-hosted
deployments can use Sentry on-premise or the reference implementation.

## Related

- [Observability](../08-runtime-architecture/18-observability.md)
- [Analytics](14-analytics.md)
- [Security Model](../08-runtime-architecture/17-security-model.md)
