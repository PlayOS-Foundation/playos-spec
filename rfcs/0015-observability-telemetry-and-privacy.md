---
rfc: 0015
title: Observability, Telemetry, and Privacy
status: Draft
authors: []
created: 2026-07-22
---

# RFC-0015: Observability, Telemetry, and Privacy

> **Status:** Draft

## Summary

This RFC defines the PlayOS observability architecture — logging,
metrics, crash reporting, and telemetry — and the privacy guarantees
that govern data collection.  It formalises the
[Observability][obs] chapter in the Runtime Architecture and the
privacy and telemetry chapters in Part XIII (Security, Privacy and
Trust).

[obs]: ../book/src/08-runtime-architecture/18-observability.md

## Motivation

A console platform needs observability to debug issues, improve
performance, and fix crashes.  But it must do so without compromising
player privacy.  Without a formal policy:

- Developers would have no visibility into how their apps perform on
  real devices.
- Crash reports might contain personal data.
- Telemetry might be collected without consent.
- The platform would violate GDPR and similar privacy regulations.

This RFC establishes the line between necessary diagnostics and
privacy-invasive surveillance.

## Guide-Level Explanation

PlayOS collects three kinds of data:

| Kind | Purpose | Who sees it | Opt-in? |
|---|---|---|---|
| **Local diagnostics** | Debug your own device | You (the player) | Always available |
| **Developer analytics** | See how your app performs | You (the developer) | App-specific opt-in |
| **Platform telemetry** | Improve PlayOS itself | PlayOS Foundation | Opt-in, anonymized |

**Nothing is collected without consent.**  Local diagnostics are always
available but never leave the device.  Developer analytics require the
player to opt in per-app.  Platform telemetry is opt-in at the system
level.

Crash reports are sanitized before upload — no save data, no
screenshots, no personal information.

## Reference-Level Explanation

### 1. Logging

#### Log levels

Applications emit logs through the Platform API:

```cpp
Log::Trace("GPU", "Shader compiled in 2.3ms");        // Dev mode only
Log::Debug("Input", "Controller hat switch: up");      // Dev mode only
Log::Info("Game", "Level 3 loaded");                    // Always
Log::Warning("Audio", "Buffer underrun on channel 2");  // Always
Log::Error("Network", "Connection timeout to server");  // Always
Log::Fatal("Engine", "Failed to create rendering context"); // Always
```

Trace and Debug levels are only recorded when developer mode is
enabled.  Info, Warning, Error, and Fatal are always recorded locally.

#### Log routing

| Destination | Content | Retention |
|---|---|---|
| Local file (`/var/log/playos/`) | All levels | 30 days, rotated |
| Developer console (SSH/ADB) | All levels (dev mode) | Stream only |
| Cloud (if telemetry opted in) | Warning, Error, Fatal only | 90 days |

Log files MUST NOT contain:
- Save data or configuration contents.
- Screenshots or frame buffer contents.
- Input sequences that could reconstruct passwords.
- Network packet payloads.
- Personal identifiers (account names, emails, IP addresses).

### 2. Metrics

The Runtime exposes metrics through the Platform API:

```cpp
auto metrics = Diagnostics::GetFrameMetrics();
// metrics.averageFrameTimeMs, metrics.frameDrops, ...
```

#### Metric categories

| Category | Metrics | Privacy |
|---|---|---|
| **Frame performance** | Frame time, frame drops, vsync misses | Non-sensitive |
| **Memory** | RSS, heap, GPU memory | App-internal only |
| **CPU** | Per-thread CPU time | App-internal only |
| **Disk I/O** | Read/write throughput, save file size | App-internal only |
| **Network I/O** | Bytes sent/received, connection count | Non-sensitive |
| **Input** | Button press count, axis usage | App-internal only |
| **Audio** | Buffer underruns, device changes | Non-sensitive |

#### Developer overlay

In developer mode, a performance HUD is available:

```text
FPS: 59.8   Frame: 16.2ms   CPU: 34%   GPU: 62%   RAM: 1.2GB
```

This is never shown in production builds.

#### Cloud metrics

When platform telemetry is opted in, aggregated metrics are uploaded:

- **Aggregate only** — per-device metrics are never uploaded.  Only
  statistical aggregates (p50, p95, p99) across the install base.
- **No per-app breakdown** — platform telemetry does not identify which
  app produced which metrics.
- **No frame-level data** — frame times are aggregated per session,
  not per frame.

### 3. Crash Reporting

#### Crash detection

When an application crashes (signal, unhandled exception, OOM):

1. The Lifecycle Manager detects the crash via `waitpid` status.
2. A **minidump** is captured (application process only — not the
   Runtime or other apps).
3. The minidump is written to `/var/log/playos/crashes/`.
4. If the player has opted in to crash reporting, the minidump is
   uploaded to the cloud.

#### Minidump contents

A minidump MUST contain only:
- Stack trace of the crashing thread.
- Register state at crash time.
- Loaded module list (executable and shared libraries, with versions).
- Platform API version and Runtime version.

A minidump MUST NOT contain:
- Heap memory contents.
- Save data or configuration.
- Input history.
- Screenshots or frame buffer contents.
- Environment variables.
- Account identifiers.

#### Crash report lifecycle

1. Minidump is uploaded (if opted in).
2. Cloud service symbolicates the stack trace.
3. Crash is grouped by signature (stack trace hash).
4. Developer dashboard shows crash groups for their apps.
5. Minidumps are deleted after 90 days.

#### Platform crashes

Runtime service crashes are reported through the same mechanism but
are always uploaded (they contain no user data).  Runtime crash
reports are visible only to Runtime maintainers.

### 4. Telemetry Policy

#### Opt-in model

Telemetry is **disabled by default**.  During first boot, the Shell
presents a privacy settings screen:

```
┌──────────────────────────────────────────┐
│  Privacy Settings                        │
├──────────────────────────────────────────┤
│                                          │
│  Help improve PlayOS                     │
│  Share anonymous usage data to help      │
│  us improve the platform.                │
│                                          │
│  [ ] Share platform telemetry            │
│  [ ] Share crash reports                 │
│                                          │
│  No personal data is ever collected.     │
│  You can change these at any time in     │
│  Settings → Privacy.                     │
└──────────────────────────────────────────┘
```

#### What is collected (when opted in)

| Data point | Purpose | Retention |
|---|---|---|
| Platform API version | Compatibility tracking | Indefinite (aggregate) |
| Runtime version | Update adoption rate | Indefinite (aggregate) |
| Device model | Hardware support prioritisation | Indefinite (aggregate) |
| Session count and duration | Engagement metrics | 90 days |
| Crash count and signatures | Stability tracking | 90 days |
| Feature usage (capabilities used) | API design decisions | 90 days |

#### What is NEVER collected

- Save data or configuration contents.
- Screenshots or frame buffer contents.
- Input sequences or button press patterns.
- Network traffic contents or destinations.
- Account identifiers, email addresses, or names.
- Precise location (IP-based country is the maximum granularity).
- Installed application list.

### 5. Data Storage and Deletion

- All cloud-stored telemetry and crash data is stored in the EU
  (Hetzner or equivalent) and encrypted at rest.
- Players can request data deletion through their PlayOS account.
- Data is automatically deleted after its retention period.
- The platform MUST provide a data export (GDPR Article 20) —
  all data associated with an account, in machine-readable format.

### 6. Developer-Facing Analytics

Separate from platform telemetry, developers can opt into per-app
analytics through the developer portal:

| Metric | Available |
|---|---|
| Install count (total) | ✅ |
| Crash rate (crashes per session) | ✅ |
| Session count and duration | ✅ (aggregate) |
| Platform API version distribution | ✅ |
| Device model distribution | ✅ |
| Per-player data | ❌ Never |

Developer analytics are aggregated and anonymized.  A developer cannot
identify individual players.

## Drawbacks

- Opt-in telemetry means the platform has less data than opt-out models
  (used by most commercial consoles).  This slows data-driven
  improvements.
- Sanitizing crash minidumps is technically challenging — heap memory
  may contain PII even if the stack trace doesn't.
- Aggregating frame-level metrics to session-level loses granularity
  that could help diagnose performance issues.
- GDPR compliance adds operational overhead (data export, deletion
  requests, DPO).

## Rationale and Alternatives

- **Opt-out telemetry (like most consoles):** More data but violates
  GDPR consent requirements and the Privacy by Default principle.
  Rejected.
- **No telemetry at all:** Minimises privacy risk but leaves the
  platform blind to real-world issues.  Opt-in balances privacy with
  platform quality.
- **Full crash dumps (including heap):** More useful for debugging but
  impossible to guarantee PII-free.  Minidumps with stack-only data are
  sufficient for crash grouping.
- **Per-app telemetry always on:** Would give developers rich analytics
  but violates player trust.  Per-app opt-in gives players control.

## Prior Art

- **Firefox Telemetry** (opt-out, documented, data dictionary): the
  gold standard for transparent telemetry.  PlayOS adopts the same
  approach of publishing exactly what is collected.
- **Apple crash reporting** (opt-in, symbolicated server-side): model
  for the crash reporting pipeline.
- **Sentry / Backtrace** (minidump-based crash aggregation): reference
  implementation for crash grouping and developer dashboards.
- **GDPR Article 25 (Data Protection by Design):** the legal framework
  that mandates opt-in, data minimization, and deletion.  PlayOS
  compliance is designed-in, not bolted-on.

## Unresolved Questions

- How is telemetry data transported?  A separate `telemetry.playos.org`
  endpoint with its own authentication, or piggybacking on the cloud
  API?
- Should the platform report which apps a player has installed
  (anonymized) to help with compatibility testing?  Leaning toward
  "no" — even anonymized app lists could be fingerprintable.
- What is the retention period for developer-facing analytics?
  Proposed: 90 days for raw data, indefinite for aggregates.

## Future Possibilities

- **Differential privacy:** Add calibrated noise to aggregate metrics
  to provide mathematical privacy guarantees beyond anonymization.
- **On-device crash analysis:** Symbolicate and group crashes on the
  device before upload, so only crash signatures leave the device.
- **Privacy dashboard:** A player-facing dashboard showing exactly what
  data has been collected and when, with one-click deletion.
- **Federated analytics:** Compute aggregates across devices without
  centralizing raw data, using federated learning techniques.
