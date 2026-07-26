# PlayOS Book — Roadmap

Last updated: 2026-07-26
Spec version: 0.4.0 (Draft)

## Overall completion

| Status | Parts | Files | ~Lines |
|---|---|---|---|
| ✅ Written | 0–12 | 181 | ~13,000 |
| ⏳ Stubs | 13–16, Appendices, 1 in Part VI | 65 | ~320 |
| **Total** | **17** | **246** | **~13,320** |

The book is approximately **74% complete** by chapter count. All core
specification content (Parts 0–12) is written with one stub (Font API)
in Part VI. The remaining 26% consists of governance, security,
certification, developer guide, and appendix chapters — all currently
placeholders.

---

## Per-part status

### ✅ Part 0 — Front matter
6/6 complete. Preface, status, reading guide, normative language,
terminology, and glossary.

### ✅ Part I — Vision
7/7 complete. What is PlayOS, mission, problem statement, non-goals,
platform promise, ecosystem vision, long-term roadmap.

Informed by RFCs: [0001](rfcs/0001-platform-principles.md)

### ✅ Part II — Platform Principles
9/9 complete. All nine platform principles documented.

Informed by RFCs: [0001](rfcs/0001-platform-principles.md)

### ✅ Part III — Platform Architecture
12/12 complete. Layered architecture, all major components, and
repository map.

### ✅ Part IV — Target Model
8/8 complete. Runtime devices, SDK targets, reference/certified
devices, platform backends, device profiles, compatibility levels.

Informed by RFCs: [0002](rfcs/0002-runtime-devices-and-sdk-targets.md),
[0006](rfcs/0006-device-profile-format.md)

### ✅ Part V — Capability Model
10/10 complete. Full capability model including required, optional,
vendor capabilities, groups, feature levels, and the capability
registry.

Informed by RFCs: [0003](rfcs/0003-capability-model.md)

### ✅ Part VI — Platform API
25/26 complete (1 stub). All API modules specified: lifecycle, input,
storage, display, audio, power, network, overlay, cloud saves,
leaderboards, achievements, marketplace, user profile, notifications,
logging, permissions, extensions, font, and API versioning.

| Chapter | Status |
|---|---|
| 01–24 | ✅ Written |
| 25 — Font API | ⏳ Stub ([RFC-0020](../rfcs/0020-typography-and-font-system.md)) |
| 26 — API Versioning | ✅ Written |

Informed by RFCs: [0004](rfcs/0004-platform-api-surface.md),
[0008](rfcs/0008-game-lifecycle-contract.md),
[0020](rfcs/0020-typography-and-font-system.md)

### ✅ Part VII — Engine Integration
9/9 complete. Engine-agnostic contract, raylib reference kit, SDL,
custom engines, Godot future, input bridging, rendering boundaries,
integration guidelines.

Informed by RFCs: [0012](rfcs/0012-engine-integration-contract.md)

### ✅ Part VIII — Runtime Architecture
18/18 complete. Boot model, Linux reference runtime, compositor model,
application lifecycle, package execution, runtime services, IPC,
input routing, game switching, overlay integration, audio/network
startup, updates, suspend/resume, security model, observability.

Informed by RFCs: [0008](rfcs/0008-game-lifecycle-contract.md),
[0013](rfcs/0013-ipc-and-runtime-service-architecture.md),
[0019](rfcs/0019-update-and-patch-distribution-model.md)

Related ADRs: [0002](adr/0002-use-wlroots-tinywl-compositor.md),
[0003](adr/0003-arch-linux-reference-runtime-base.md),
[0004](adr/0004-use-alpine-linux-reference-os-base.md)

### ✅ Part IX — Shell and UX
16/16 complete. Console UX principles, controller-first navigation,
touch, home screen, library, store, settings, quick settings,
downloads, notifications, power menu, developer mode, accessibility,
themes, reference shell.

Informed by RFCs: [0018](rfcs/0018-accessibility-as-a-platform-requirement.md)

### ✅ Part X — Package Format
16/16 complete. Package goals, anatomy, manifest, metadata, assets,
executables, ABIs, permissions, signing, install/uninstall, updates,
release channels, dev packages, validation.

Informed by RFCs: [0005](rfcs/0005-package-format.md),
[0019](rfcs/0019-update-and-patch-distribution-model.md)

### ✅ Part XI — Cloud and Marketplace
19/19 complete. Self-hosting principles, cloud architecture, accounts,
developer portal, package storage, marketplace catalog, store sources,
entitlements, payments, cloud saves, leaderboards, achievements,
analytics, crash reporting, updates/CDN, federation, OEM stores,
API versioning.

Informed by RFCs: [0007](rfcs/0007-cloud-and-marketplace-architecture.md),
[0016](rfcs/0016-self-hosting-and-store-federation.md)

### ✅ Part XII — Device Model and Porting
18/18 complete. Device profile schema, backend/input/display/audio/
power/storage/network porting, runtime device and SDK target bring-up,
ROG Ally, generic Linux PC, Orange Pi references, Windows/Android/PS
Vita SDK targets, future device checklist.

Informed by RFCs: [0006](rfcs/0006-device-profile-format.md),
[0014](rfcs/0014-device-porting-and-bring-up-model.md)

Related ADRs: [0003](adr/0003-arch-linux-reference-runtime-base.md),
[0004](adr/0004-use-alpine-linux-reference-os-base.md)

---

### ⏳ Part XIII — Security, Privacy and Trust
**11/11 stubs.** Chapter outline defined, content needed.

| # | Chapter | Status |
|---|---|---|
| 01 | Overview | Stub |
| 02 | Trust Model | Stub |
| 03 | Package Signing | Stub |
| 04 | Permissions | Stub |
| 05 | Sandboxing | Stub |
| 06 | Secure Updates | Stub |
| 07 | Account Security | Stub |
| 08 | Privacy Principles | Stub |
| 09 | Telemetry Policy | Stub |
| 10 | Child Safety and Parental Controls | Stub |
| 11 | Supply Chain Security | Stub |

Related RFCs: [0009](rfcs/0009-security-sandboxing-trust-model.md),
[0015](rfcs/0015-observability-telemetry-and-privacy.md)

### ⏳ Part XIV — Certification
**12/12 stubs.** Chapter outline defined, content needed.

| # | Chapter | Status |
|---|---|---|
| 01 | Overview | Stub |
| 02 | Certification Levels | Stub |
| 03 | Platform API Compatible | Stub |
| 04 | Runtime Compatible | Stub |
| 05 | PlayOS Certified Hardware | Stub |
| 06 | Game Certification | Stub |
| 07 | Store Certification | Stub |
| 08 | Device Certification Tests | Stub |
| 09 | API Conformance Tests | Stub |
| 10 | Performance Requirements | Stub |
| 11 | Accessibility Requirements | Stub |
| 12 | Certification Process | Stub |

Related RFCs: [0010](rfcs/0010-certification-and-conformance-program.md),
[0018](rfcs/0018-accessibility-as-a-platform-requirement.md)

### ⏳ Part XV — Developer Guide
**18/18 stubs.** Chapter outline defined, content needed.

| # | Chapter | Status |
|---|---|---|
| 01 | Overview | Stub |
| 02 | Installing the SDK | Stub |
| 03 | Your First PlayOS App | Stub |
| 04 | Raylib Starter Project | Stub |
| 05 | Using Capabilities | Stub |
| 06 | Handling Input | Stub |
| 07 | Saving Data | Stub |
| 08 | Cloud Saves | Stub |
| 09 | Leaderboards | Stub |
| 10 | Achievements | Stub |
| 11 | Packaging Your Game | Stub |
| 12 | Testing on Linux | Stub |
| 13 | Testing on Android | Stub |
| 14 | Testing on PS Vita | Stub |
| 15 | Publishing to a Store | Stub |
| 16 | Debugging | Stub |
| 17 | Performance Guidelines | Stub |
| 18 | Release Checklist | Stub |

Related RFCs: [0017](rfcs/0017-developer-experience-and-sdk-model.md)

### ⏳ Part XVI — Governance and Process
**12/12 stubs.** Chapter outline defined, content needed.

| # | Chapter | Status |
|---|---|---|
| 01 | Overview | Stub |
| 02 | PlayOS Foundation | Stub |
| 03 | Governance Model | Stub |
| 04 | RFC Process | Stub |
| 05 | ADR Process | Stub |
| 06 | Versioning Policy | Stub |
| 07 | Compatibility Policy | Stub |
| 08 | Deprecation Policy | Stub |
| 09 | Release Process | Stub |
| 10 | Contribution Model | Stub |
| 11 | Community Guidelines | Stub |
| 12 | Commercial Ecosystem | Stub |

Related RFCs: [0011](rfcs/0011-governance-versioning-compatibility.md)

### ⏳ Appendices
**11/11 stubs.** Chapter outline defined, content needed.

| # | Chapter | Status |
|---|---|---|
| 01 | API Index | Stub |
| 02 | Capability Registry | Stub |
| 03 | Error Code Registry | Stub |
| 04 | Manifest Schema | Stub |
| 05 | Device Profile Schema | Stub |
| 06 | Permission Registry | Stub |
| 07 | Package Signature Format | Stub |
| 08 | CLI Reference | Stub |
| 09 | Glossary | Stub |
| 10 | FAQ | Stub |
| 11 | Bibliography | Stub |

---

## Writing priority

Recommended order for completing remaining parts, based on dependency
and near-term implementation needs:

### Priority 1 — Near-term (before Phase 1 completion)
These are needed for Phase 1 (Platform Foundation) and Phase 2 (Cloud
and Marketplace) implementation:

1. **Part XIII — Security, Privacy and Trust** — needed for any
   production deployment. Trust model, sandboxing, and permissions
   directly inform runtime implementation.
2. **Part XVI — Governance and Process** — needed for accepting external
   contributions. RFC/ADR processes, versioning, and compatibility
   policies are already followed in practice.
3. **Appendices: Glossary, FAQ** — small wins that improve book
   usability immediately.

### Priority 2 — Mid-term (before Phase 3)
4. **Part XIV — Certification** — needed before any hardware or game
   certification program launches.
5. **Part XV — Developer Guide** — needed before onboarding external
   developers in Phase 3.
6. **Appendices: API Index, Capability Registry, Error Code Registry,
   Manifest Schema, Device Profile Schema** — reference material that
   gets value once the corresponding specification parts stabilize.

### Priority 3 — Pre-1.0
7. **Appendices: Permission Registry, Package Signature Format, CLI
   Reference, Bibliography** — final polish before 1.0.

---

## RFC coverage

Existing RFCs provide design rationale for most parts of the
specification. Each RFC is linked to the book part(s) it informs:

| RFC | Topic | Book Parts |
|---|---|---|
| [0001](rfcs/0001-platform-principles.md) | Platform Principles | [Part I](#-part-i--vision), [Part II](#-part-ii--platform-principles) |
| [0002](rfcs/0002-runtime-devices-and-sdk-targets.md) | Devices and Targets | [Part IV](#-part-iv--target-model) |
| [0003](rfcs/0003-capability-model.md) | Capability Model | [Part V](#-part-v--capability-model) |
| [0004](rfcs/0004-platform-api-surface.md) | Platform API Surface | [Part VI](#-part-vi--platform-api) |
| [0005](rfcs/0005-package-format.md) | Package Format | [Part X](#-part-x--package-format) |
| [0006](rfcs/0006-device-profile-format.md) | Device Profiles | [Part IV](#-part-iv--target-model), [Part XII](#-part-xii--device-model-and-porting) |
| [0007](rfcs/0007-cloud-and-marketplace-architecture.md) | Cloud/Marketplace | [Part XI](#-part-xi--cloud-and-marketplace) |
| [0008](rfcs/0008-game-lifecycle-contract.md) | Game Lifecycle | [Part VI](#-part-vi--platform-api), [Part VIII](#-part-viii--runtime-architecture) |
| [0009](rfcs/0009-security-sandboxing-trust-model.md) | Security | [Part XIII](#-part-xiii--security-privacy-and-trust) ⏳ |
| [0010](rfcs/0010-certification-and-conformance-program.md) | Certification | [Part XIV](#-part-xiv--certification) ⏳ |
| [0011](rfcs/0011-governance-versioning-compatibility.md) | Governance | [Part XVI](#-part-xvi--governance-and-process) ⏳ |
| [0012](rfcs/0012-engine-integration-contract.md) | Engine Integration | [Part VII](#-part-vii--engine-integration) |
| [0013](rfcs/0013-ipc-and-runtime-service-architecture.md) | IPC/Runtime | [Part VIII](#-part-viii--runtime-architecture) |
| [0014](rfcs/0014-device-porting-and-bring-up-model.md) | Porting | [Part XII](#-part-xii--device-model-and-porting) |
| [0015](rfcs/0015-observability-telemetry-and-privacy.md) | Observability & Privacy | [Part XIII](#-part-xiii--security-privacy-and-trust) ⏳ |
| [0016](rfcs/0016-self-hosting-and-store-federation.md) | Self-hosting | [Part XI](#-part-xi--cloud-and-marketplace) |
| [0017](rfcs/0017-developer-experience-and-sdk-model.md) | Developer Guide | [Part XV](#-part-xv--developer-guide) ⏳ |
| [0018](rfcs/0018-accessibility-as-a-platform-requirement.md) | Accessibility | [Part IX](#-part-ix--shell-and-ux) ✅, [Part XIV](#-part-xiv--certification) ⏳ |
| [0019](rfcs/0019-update-and-patch-distribution-model.md) | Updates | [Part VIII](#-part-viii--runtime-architecture), [Part X](#-part-x--package-format) |
| [0020](rfcs/0020-typography-and-font-system.md) | Typography & Font System | [Part VI](#-part-vi--platform-api) ⏳ |

All 20 RFCs are accepted. Parts XIII–XVI and Appendices have RFCs
ready to be turned into specification prose.

---

## ADR decisions

| ADR | Decision | Relevant Book Part |
|---|---|---|
| [0001](adr/0001-use-mdbook-for-the-playos-book.md) | Use mdBook for the PlayOS Book | Part 0 (how the book is built) |
| [0002](adr/0002-use-wlroots-tinywl-compositor.md) | Use wlroots/tinywl for compositor | [Part VIII](#-part-viii--runtime-architecture) |
| [0003](adr/0003-arch-linux-reference-runtime-base.md) | Arch Linux as reference runtime base | [Part VIII](#-part-viii--runtime-architecture), [Part XII](#-part-xii--device-model-and-porting) |
| [0004](adr/0004-use-alpine-linux-reference-os-base.md) | Alpine Linux as reference OS base | [Part VIII](#-part-viii--runtime-architecture), [Part XII](#-part-xii--device-model-and-porting) |

---

## How to contribute

See [CONTRIBUTING.md](CONTRIBUTING.md) for the full process.

Quick start for writing a stub chapter:
1. Read the corresponding RFC (linked above).
2. Remove the stub marker and write the chapter content.
3. Follow the style guide in [AGENTS.md](AGENTS.md).
4. Update this ROADMAP.md to mark the chapter as done.
