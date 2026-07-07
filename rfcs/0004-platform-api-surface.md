---
rfc: 0004
title: Platform API Surface
status: Accepted
authors: []
created: 2026-01-01
---

# RFC-0004: Platform API Surface

> **Status:** Accepted

## Summary

This RFC defines the partition of the PlayOS Platform API into a portable
core subset (available on all conformant targets, including SDK Targets) and
runtime-only features (available only on Runtime Devices). It sets the
boundary and the rules for crossing it, referencing RFC-0002 (target model)
and RFC-0003 (capabilities).

## Motivation

The Platform API must be usable on Windows, macOS, Linux desktop, Android,
and PS Vita without the PlayOS Runtime. But it must also expose console-grade
features — overlay, game switching, package installation — to devices that
do run the full runtime. A single undifferentiated surface would either force
every target to implement the runtime (excluding SDK targets) or remove the
console features (removing what makes PlayOS a console). A deliberate
partition resolves both.

## Guide-Level Explanation

Applications include `playos/playos.h` and call the same API everywhere.
Whether a feature exists is discovered at runtime via capabilities, not at
compile time.

**Core (portable) subset** — always present:

- `Lifecycle` (`Init` / `Update` / `Shutdown`)
- `Input` (logical buttons, axes, `Down` / `Pressed` / `GetAxis`)
- `Capabilities` (`Has` / `List` / `Id`)
- `Storage` (save and config paths)
- `Display` (display size, safe area)
- `Audio` (volume, basic playback helper)

**Runtime-only** — available on Runtime Devices, discovered via capabilities:

- `Overlay` (system overlay surface)
- `Marketplace` (store client API)
- `CloudSave` / `Leaderboard` / `Achievements` (cloud-backed)
- `System` (suspend/resume, package installation, brightness/TDP/fan)
- `Notification` (system notification surface)

## Reference-Level Explanation

Rules:

- A module proposed for the core subset MUST be implementable on every
  reference SDK target. If not, it belongs in runtime-only.
- A runtime-only module MUST be gated by a capability. A target that cannot
  provide it simply reports the capability absent; the API still compiles
  and links but returns well-defined errors.
- A new module MUST live in a single public header under `include/playos/`
  whose name matches the module (e.g. `include/playos/storage.h` for
  `PlayOS::Storage`).
- The umbrella header `include/playos/playos.h` includes all public
  module headers.

## Drawbacks

- The partition is a design constraint; some features that feel natural on
  a console require a capability query on SDK targets.
- A large runtime-only tier could make the core subset feel sparse.
  Mitigation: the core subset is deliberately small for a low-friction
  on-ramp on SDK targets.

## Rationale and Alternatives

- **One surface, no distinction.** Rejected: forces every target to
  implement the runtime (Vita cannot run the compositor).
- **Per-platform API variants.** Rejected: fragments the platform and
  requires conditional compilation in application code.

## Unresolved Questions

- The exact module list is refined as new modules are added. This RFC
  records the partition rule; the book chapters list current modules.
- Whether runtime features deserve a partial presence on SDK targets
  (e.g. achievements as local-only metadata) is deferred.
