---
rfc: 0010
title: Certification and Conformance Program
status: Draft
authors: []
created: 2026-07-22
---

# RFC-0010: Certification and Conformance Program

> **Status:** Draft

## Summary

This RFC defines the PlayOS certification program: three conformance
levels, the certification process, conformance test suites, performance
and accessibility requirements, and the certification lifecycle.  It
provides the foundation for Part XIV (Certification) of the PlayOS Book
and defines what it means for a device, application, or store to be
"PlayOS Compatible" or "PlayOS Certified."

## Motivation

Without a certification program, "PlayOS Compatible" has no meaning.
A device vendor could claim compatibility while shipping a broken
implementation; a game could claim to support PlayOS while using
platform-specific hacks.  Certification:

- Gives developers confidence that their app will run on certified
  devices.
- Gives players confidence that a certified device delivers the PlayOS
  experience.
- Gives hardware vendors a clear target to implement against.
- Enables the PlayOS trademark to mean something concrete.

## Guide-Level Explanation

There are three levels of PlayOS conformance:

| Level | Who it's for | What it means |
|---|---|---|
| **Platform API Compatible** | SDK Targets (Windows, macOS, Android) | The API works; apps using only required capabilities run correctly |
| **Runtime Compatible** | Runtime Devices (handhelds, consoles) | Full PlayOS experience: overlay, game switching, package install |
| **PlayOS Certified Hardware** | Consumer devices | Everything above + performance, accessibility, security review |

A developer targeting "Platform API Compatible" knows their app will run
on any PlayOS device.  A developer needing overlay or cloud saves targets
"Runtime Compatible."  A hardware vendor aiming for "PlayOS Certified"
knows exactly what tests they must pass.

## Reference-Level Explanation

### 1. Certification Levels

#### Level 1: Platform API Compatible

The minimum bar.  The target:

- MUST implement all **required capabilities** as defined in the
  Capability Registry.
- MUST pass the **API Conformance Test Suite** (see §3).
- MUST implement `Lifecycle::Init/Update/Shutdown` correctly.
- MUST implement the Input API with at least the required button set.
- MUST implement the Storage API with per-app save directories.

This level does NOT require: overlay, game switching, suspend/resume,
package installation, cloud features, or a compositor.  It is the level
for SDK Targets and lightweight ports.

#### Level 2: Runtime Compatible

The console experience.  In addition to Level 1:

- MUST implement all **runtime capabilities**: `system.overlay`,
  `system.game-switching`, `system.package-install`, `system.suspend-resume`.
- MUST pass the **Runtime Conformance Test Suite**.
- MUST implement the full Package Execution contract (RFC-0008).
- MUST implement the IPC bus and all required Runtime Services.
- MUST implement the Security Model (RFC-0009): sandboxing, permissions,
  code signing verification.
- Cloud capabilities (`cloud.saves`, `cloud.leaderboards`,
  `cloud.achievements`) MUST be present when connected to a PlayOS
  Cloud instance.

#### Level 3: PlayOS Certified Hardware

The highest bar.  In addition to Level 2:

- MUST pass **performance requirements** (§4).
- MUST pass **accessibility requirements** (§5).
- MUST pass **security review** (§6).
- MUST meet **physical design guidelines** (controller layout, button
  labeling).
- MUST ship with a verified boot chain and hardware root of trust.
- MUST include the PlayOS Shell as the primary user interface.

### 2. Capability Matrix

| Capability | Level 1 | Level 2 | Level 3 |
|---|---|---|---|
| Required capabilities (all) | ✅ | ✅ | ✅ |
| `system.overlay` | ❌ | ✅ | ✅ |
| `system.game-switching` | ❌ | ✅ | ✅ |
| `system.package-install` | ❌ | ✅ | ✅ |
| `system.suspend-resume` | ❌ | ✅ | ✅ |
| Cloud capabilities | ❌ | ✅ (when connected) | ✅ (when connected) |
| Performance targets | ❌ | ❌ | ✅ |
| Accessibility features | ❌ | ❌ | ✅ |
| Security review | ❌ | ❌ | ✅ |
| Physical design review | ❌ | ❌ | ✅ |

### 3. Conformance Test Suites

#### API Conformance Test Suite (ACTS)

The ACTS is a cross-platform test harness that verifies Platform API
behaviour:

- Every public function is called with valid and invalid inputs.
- Return values are checked against the specification.
- Error codes are verified for known failure modes.
- Capability queries return expected values for the target profile.
- Input, storage, display, and lifecycle APIs are tested end-to-end.

The ACTS MUST be runnable on the target without modification.  It is
distributed as source code (C++17, CMake) and produces a pass/fail
report in JUnit XML format.

#### Runtime Conformance Test Suite (RCTS)

The RCTS verifies Runtime behaviour beyond the API surface:

- Package installation, verification, and removal.
- Application launch, game switching, and exit (RFC-0008 contract).
- Overlay activation and deactivation.
- Suspend and resume (where `system.suspend-resume` is present).
- Permission enforcement (denied permissions block API calls).
- Sandboxing (filesystem and process isolation).

#### Certification Test Harness

A physical test harness for Level 3 certification:

- Automated input latency measurement (hardware-timed button-to-screen).
- Frame rate measurement under load.
- Boot time measurement (UEFI handoff to first interactive frame).
- Audio latency measurement.
- Battery life measurement under defined workloads.

### 4. Performance Requirements (Level 3)

| Metric | Requirement | Measurement |
|---|---|---|
| Cold boot to interactive shell | ≤ 8 seconds | UEFI handoff to first frame |
| Resume to interactive shell | ≤ 3 seconds | Power button to first frame |
| Frame rate (Shell UI) | ≥ 60 FPS sustained | 60-second measurement window |
| Input latency (button to action) | ≤ 16 ms (1 frame at 60 Hz) | Hardware-timed |
| Game launch time (cold) | ≤ 5 seconds | Select to first game frame |
| Game switching time | ≤ 2 seconds | Home press to Shell interactive |
| Audio output latency | ≤ 20 ms | Signal generator to headphone jack |

### 5. Accessibility Requirements (Level 3)

The device MUST provide:

| Feature | Requirement |
|---|---|
| Screen reader | Controller-operated; reads all Shell UI elements |
| Button remapping | System-wide; per-application profiles |
| High contrast mode | Contrast ratio ≥ 7:1 |
| Large text mode | Minimum 36 px; scales all Shell text by 1.5× |
| Subtitle support | System-wide preferences (size, color, background) |
| Stick dead zone adjustment | Customizable per stick |
| Vibration control | Reduce or disable haptic feedback |

Color blind modes and mono audio are RECOMMENDED but not required.

### 6. Security Review (Level 3)

A third-party security review MUST verify:

- Verified boot chain is intact and cannot be bypassed.
- Root filesystem is read-only in production.
- Applications are properly sandboxed (filesystem, network, process
  isolation verified by penetration test).
- Package signatures are verified before installation.
- Permissions are enforced at the API level.
- No debug interfaces (SSH, ADB) are enabled in production.
- Secure storage for keys and credentials is implemented.

### 7. Certification Process

#### Self-attestation (Level 1)

The implementer runs the ACTS, verifies all tests pass, and submits
the test report to the PlayOS Foundation.  No fee.  The implementer
may use the "Platform API Compatible" mark.

#### Formal certification (Levels 2 and 3)

1. **Application** — implementer submits device specifications, test
   reports (ACTS + RCTS), and certification fee.
2. **Review** — Foundation reviews test reports.  For Level 3, a
   physical device is shipped to the Foundation for independent testing.
3. **Testing** — Foundation runs ACTS + RCTS independently.  For Level
   3, the certification test harness is used.
4. **Decision** — pass/fail within 30 days of receiving the device.
   Failures include a report of which tests failed.
5. **Certification grant** — successful devices are listed in the
   PlayOS Certified Device Registry.  The implementer may use the
   "PlayOS Certified" mark.
6. **Recertification** — required for each new MAJOR Specification
   version.  MINOR and PATCH updates do not require recertification.

#### Certification fee

The fee covers the cost of independent testing.  It is set annually by
the Foundation and published.  The fee is waived for open-source
hardware projects and non-profit deployments.

### 8. Certification Lifecycle

- Certification is valid for the Specification MAJOR version it was
  granted for, plus two years.
- If a security vulnerability is discovered in a certified device, the
  implementer has 90 days to patch it or certification is suspended.
- Certification may be revoked if the device is found to violate the
  conformance requirements after grant.

### 9. Game and Store Certification

#### Game certification

"PlayOS Compatible" for games is self-attested: the developer asserts
the app uses only the Platform API and declares capabilities correctly.
The marketplace MAY require ACTS pass before publishing.

"PlayOS Certified" for games requires:
- ACTS pass on at least one Level 3 device.
- Compliance with age rating requirements (ESRB/PEGI).
- No platform-specific hacks or capability circumvention.

#### Store certification

"PlayOS Compatible Store" requires:
- `.gpk` package format compliance.
- Package signing verification.
- Entitlement tracking per RFC-0007.
- Federation support (publishing `federation.json`).

## Drawbacks

- Certification adds cost and time for hardware vendors.  Small
  open-source projects may find the Level 3 process burdensome.
- Self-attestation at Level 1 has limited value — it relies on the
  implementer's honesty.
- The certification test harness requires specialized hardware
  (high-speed camera for input latency) that the Foundation must
  acquire and maintain.
- Game certification is optional, which may lead to inconsistent
  quality in the marketplace.

## Rationale and Alternatives

- **Single certification level (all-or-nothing):** Simpler but excludes
  SDK Targets and lightweight ports.  Three levels maps to the existing
  Target Model (SDK Target, Runtime Device, Certified Hardware).
- **No certification (open self-attestation only):** Reduces friction
  but makes "PlayOS" meaningless as a quality signal.  Rejected —
  console platforms need certification to maintain trust.
- **Foundation-managed test lab vs. third-party labs:** Foundation-
  managed ensures consistency but limits throughput.  Third-party
  authorized labs are a future possibility (see Future Possibilities).
- **Per-MINOR recertification:** Would be too burdensome.  Recertifying
  only at MAJOR boundaries balances assurance with practicality.

## Prior Art

- **Android Compatibility Program** (CDD + CTS + CTS Verifier): the
  closest analogue.  PlayOS adopts a similar model: specification
  (CDD equivalent), conformance test suite (CTS equivalent), and
  certification levels.
- **Vulkan Conformance Test Suite**: open-source, cross-platform,
  self-run.  PlayOS ACTS follows this model.
- **Nintendo/Microsoft/Sony platform certification**: closed,
  proprietary, expensive.  PlayOS aims for the same rigor with open
  process.
- **Wi-Fi Alliance certification**: third-party labs, branded mark,
  interoperability focus.  Reference for how an open standard can have
  meaningful certification.

## Unresolved Questions

- What is the initial certification fee?  Depends on testing costs —
  should be determined after the first reference device goes through
  the process.
- Should ACTS be runnable in CI (headless) for SDK Targets that lack
  input hardware?  Mock input backends may be needed.
- How are certification test harness hardware specifications defined?
  High-speed camera, input latency measurement device — these need
  procurement and calibration procedures.
- Should there be a "PlayOS Ready" pre-certification for devices in
  development?

## Future Possibilities

- **Authorized test labs:** Third-party labs certified to perform Level
  3 testing, increasing throughput for commercial device launches.
- **Continuous certification:** Automated ACTS + RCTS runs on every
  Runtime commit; devices that regress are flagged automatically.
- **Game certification tiers:** "PlayOS Certified Gold" for games that
  meet additional quality standards (no crashes, consistent frame rate).
- **Accessibility certification:** Separate mark for devices that exceed
  minimum accessibility requirements (e.g., support for switch
  controllers, eye tracking).
- **Cloud service certification:** Certify self-hosted cloud instances
  for interoperability.
