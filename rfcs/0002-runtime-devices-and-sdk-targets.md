---
rfc: 0002
title: Runtime Devices and SDK Targets
status: Accepted
authors: []
created: 2026-01-01
---

# RFC-0002: Runtime Devices and SDK Targets

> **Status:** Accepted

## Summary

PlayOS defines two categories of target: **Runtime Devices**, which run the
full PlayOS environment (runtime, shell, compositor, package execution, and
marketplace client), and **SDK Targets**, which support the PlayOS Platform
API (or a subset) without running the PlayOS runtime. This split lets PlayOS
serve both dedicated console-like hardware and lightweight, portable SDK
integrations from the same Platform API.

## Motivation

PlayOS must run as a complete console environment on dedicated hardware, but
it must also let developers build and test on their existing machines and let
games reach platforms that already have their own operating system. Forcing
every target to run the full runtime would exclude Windows, macOS, Android,
and homebrew consoles. Treating everything as merely an SDK would abandon the
console experience. The two-category model captures both without weakening
either.

## Guide-Level Explanation

**Runtime Devices** boot into PlayOS and provide the whole experience:
compositor-owned display, the shell, package installation and execution,
system overlay, game switching, suspend/resume, and device controls such as
power and brightness where the hardware supports them.

Reference runtime devices:

- ROG Ally (primary handheld, AMD Radeon 780M),
- Generic x86_64 PC,
- Orange Pi 5 (RK3588, ARM64).

**SDK Targets** support the portable subset of the Platform API without the
PlayOS runtime:

- Windows, macOS, Linux desktop (development),
- Android (mobile),
- PS Vita homebrew (retro/homebrew, via VitaSDK).

A game written against the Platform API runs on both categories; the
difference in available features is discovered through capabilities
(RFC-0003), not by assuming a specific target.

## Reference-Level Explanation

The Platform API surface is partitioned:

- **Core (portable) subset** — available on all conformant targets:
  input, storage (save/config paths), display information, audio helpers,
  and application lifecycle basics.
- **Runtime-only features** — available only on runtime devices:
  system overlay, package installation and the marketplace client,
  launching other applications, OS-level suspend/resume, and device power
  and brightness controls.

Rules:

- An SDK target MUST implement the core subset to be conformant at the
  "Platform API Compatible" certification level.
- A runtime device MUST additionally provide runtime-only features to be
  "Runtime Compatible."
- Applications MUST NOT assume runtime-only features exist; they MUST query
  the relevant capability first.
- The presence of a runtime-only feature is always expressed as a capability,
  so the same binary degrades gracefully on SDK targets.

These categories map directly onto the certification levels defined in the
certification part, and onto the device profile and backend model in the
device-model part.

## Drawbacks

- Maintaining a clear boundary between the core subset and runtime-only
  features requires ongoing discipline as new APIs are added.
- Developers targeting only SDK targets may be surprised that some features
  are unavailable; documentation and capability queries must make this clear.

## Rationale and Alternatives

- **Single category (everything runs the runtime).** Rejected: excludes
  Windows, macOS, Android, and homebrew consoles.
- **Single category (everything is just an SDK).** Rejected: gives up the
  console experience and runtime-level features that differentiate PlayOS.
- **Per-platform special cases.** Rejected: does not scale and encourages
  platform checks instead of capability queries.

The two-category model is the minimum structure that supports both dedicated
console hardware and broad SDK reach while keeping one Platform API.

## Unresolved Questions

- The exact list of APIs in the core subset versus runtime-only features is
  refined alongside RFC-0004 (Platform API Surface).
- Whether a "partial runtime" tier (some but not all runtime features) is
  worth defining, or whether capabilities alone are sufficient, is left open.
