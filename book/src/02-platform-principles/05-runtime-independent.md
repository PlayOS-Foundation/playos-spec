# Runtime Independent

PlayOS applications are **runtime-independent**: the same application
binary runs on a full PlayOS Runtime Device and on an SDK Target
(Windows, macOS, Linux desktop) without modification.

## The principle

> The Platform API is the contract. The Runtime is an implementation detail.

An application written to the Platform API works everywhere the Platform
API is implemented — whether that is a dedicated PlayOS console running
the full Runtime, or a developer's Windows laptop running just the SDK
target backend.

## Two tiers, one API

```text
Runtime Device (full console)
    ├── Platform API (full — all capabilities available)
    ├── Runtime services (overlay, game switching, package management)
    └── Shell (controller-first UI)

SDK Target (development)
    ├── Platform API (core subset — required capabilities only)
    ├── No runtime services
    └── No shell (application provides its own UI)
```

The Platform API is the same on both. Runtime-only features are gated by
optional capabilities. On an SDK Target, `Capabilities::Has(Overlay)`
returns `false`; the application adapts or falls back.

## Why this matters

- **Development** — develop on a desktop with fast iteration. Test on
  a device when ready. No recompilation.
- **Porting** — adding a new SDK Target (Android, PS Vita) means writing
  a backend, not rewriting the application.
- **Certification** — the same binary submitted for certification on a
  Runtime Device is the one tested on SDK Targets.

## What applications must do

- Query capabilities before using optional features.
- Provide fallback paths for absent capabilities.
- Never assume the Runtime is present.

## Related

- [Target Model](../04-target-model/01-overview.md)
- [SDK Targets](../04-target-model/03-sdk-targets.md)
- [Runtime Devices](../04-target-model/02-runtime-devices.md)
- RFC-0002 (Runtime Devices and SDK Targets)
