---
rfc: 0011
title: Governance, Versioning, and Compatibility Model
status: Draft
authors: []
created: 2026-07-22
---

# RFC-0011: Governance, Versioning, and Compatibility Model

> **Status:** Draft

## Summary

This RFC defines how the PlayOS project is governed, how its artifacts
are versioned, what compatibility guarantees are made, how deprecation
and releases are managed, how contributors participate, and how the
commercial ecosystem operates.  It provides the foundation for Part XVI
(Governance and Process) of the PlayOS Book.

## Motivation

PlayOS spans 12 repositories, multiple independent versioned artifacts
(Platform API, Runtime, Shell, Cloud, Spec), and involves contributors
from different organisations.  Without a shared governance and versioning
model:

- Breaking changes could propagate without warning.
- Contributors would not know how decisions are made.
- Hardware vendors and store operators would not know what "compatible"
  means long-term.
- The commercial ecosystem would lack clarity on what is free vs. paid.

This RFC establishes the rules before the project reaches v1.0.

## Guide-Level Explanation

**For developers:** The Platform API version is `MAJOR.MINOR.PATCH`.
Your app compiled against `1.2.0` runs on any device with Platform API
`1.x` (where `x >= 2`).  Before we remove a function, we mark it
deprecated for at least one MINOR release — you get compiler warnings
and time to migrate.

**For contributors:** Major decisions go through an RFC.  Implementation
decisions go through an ADR.  Both are public, reviewed, and merged
after consensus.  The PlayOS Foundation stewards the project but does
not control it — maintainers are selected by contribution merit.

**For hardware vendors:** "PlayOS Compatible" is defined by the
specification at a specific version.  You implement that version.  New
versions are additive within a MAJOR — you can upgrade without breaking
existing apps.

## Reference-Level Explanation

### 1. Multi-Repository Versioning

Each artifact in the PlayOS ecosystem is versioned independently:

| Artifact | Versioning | Repository |
|---|---|---|
| PlayOS Specification | `MAJOR.MINOR.PATCH` | `playos-spec` |
| Platform API | `MAJOR.MINOR.PATCH` | `playos-platform-api` |
| Runtime | `MAJOR.MINOR.PATCH` | `playos-runtime` |
| Shell | `MAJOR.MINOR.PATCH` | `playos-shell` |
| Cloud Services | `MAJOR.MINOR.PATCH` per service | `playos-cloud` |
| Marketplace | `MAJOR.MINOR.PATCH` | `playos-marketplace` |
| Tools / CLI | `MAJOR.MINOR.PATCH` | `playos-tools` |
| Reference OS | Date-based: `YYYY.MM.PATCH` | `playos-refdistro` |
| Device Profiles | `MAJOR.MINOR` | `playos-reference-devices` |

The Specification version is the **platform epoch**.  All other
artifacts declare which specification version they implement:

```json
// In Platform API's playos.h
#define PLAYOS_SPEC_VERSION_MAJOR 1
#define PLAYOS_SPEC_VERSION_MINOR 0
```

#### Platform API versioning rules

| Bump | When |
|---|---|
| **MAJOR** | Symbol removed/renamed, signature changed, enum reordered, required capability removed |
| **MINOR** | New module, function, enum, capability, or backend added |
| **PATCH** | Bug fix, doc update, no API surface change |

These rules are already specified in the [API Versioning][api-ver]
chapter and are normative.

#### Runtime and Shell versioning

The Runtime and Shell follow the same `MAJOR.MINOR.PATCH` scheme but
with different semantics:

| Bump | Runtime / Shell |
|---|---|
| **MAJOR** | Breaking change to IPC protocol, compositor protocol, or package execution contract |
| **MINOR** | New runtime service, new Shell screen, new overlay feature |
| **PATCH** | Bug fix, performance improvement |

#### Specification versioning

| Bump | Specification |
|---|---|
| **MAJOR** | Breaking change to the platform model (e.g., new target category, removed capability) |
| **MINOR** | New part, new chapter, new capability registered, new RFC accepted |
| **PATCH** | Clarification, typo fix, non-semantic change |

[api-ver]: ../book/src/06-platform-api/25-api-versioning.md

### 2. Compatibility Guarantees

#### Platform API compatibility

Within a MAJOR version, the Platform API guarantees:

1. **Source compatibility** — code compiled against `1.N` compiles
   against `1.N+K` without changes.
2. **Binary compatibility** — a binary linked against `1.N` loads and
   runs against `1.N+K` without recompilation.
3. **Behavioural compatibility** — documented semantics do not change
   within a MAJOR version.

These guarantees apply for at least **two years** after the next MAJOR
version is released.

#### Package format compatibility

A `.gpk` package produced for Runtime `1.N` MUST install and run on
Runtime `1.N+K`.  The Runtime is responsible for any format migration.

#### Specification compatibility

A device certified against Specification `1.N` remains certified through
`1.N+K` — no recertification is required for MINOR or PATCH updates.

#### What is NOT covered

- **Undocumented behaviour** — if not in the specification, it may change.
- **Experimental APIs** — marked `PLAYOS_EXPERIMENTAL` in headers.
- **Engine APIs** — the engine is the developer's choice; PlayOS does
  not guarantee engine compatibility.
- **Vendor extensions** — `vendor.*` capabilities and extensions may
  change at the vendor's discretion.

### 3. Deprecation Lifecycle

Before a public symbol is removed:

1. **Deprecation announcement** (MINOR release `N`):
   - Symbol marked with `[[deprecated]]` attribute.
   - Deprecation documented in release notes.
   - Replacement API documented (if applicable).
2. **Deprecation warning period** (at least one full MINOR cycle):
   - Compiler emits deprecation warning.
   - Symbol continues to function normally.
3. **Removal** (next MAJOR release):
   - Symbol removed from headers and library.
   - Removal documented in migration guide.

Capabilities follow a different rule: capability identifiers are
**immutable**.  If a capability's semantics must change, a new
identifier is created and the old one is deprecated but never removed.

### 4. Release Process

#### Release cadence

| Artifact | Cadence | Notes |
|---|---|---|
| Specification | 2× per year (aligned with MINOR) | MAJOR releases are rare (years apart) |
| Platform API | Aligned with Specification MINOR | PATCH releases as needed |
| Runtime | Aligned with Platform API MINOR | PATCH releases as needed |
| Shell | Aligned with Runtime MINOR | PATCH releases as needed |
| Cloud Services | Independent, continuous | Each service has its own release cycle |

#### Release checklist

For a MINOR release of the Platform API:

1. All changes merged to `main`.
2. API conformance tests pass on all reference devices.
3. Deprecation warnings documented in `CHANGELOG.md`.
4. `PLAYOS_VERSION_MINOR` bumped in `playos/playos.h`.
5. Tagged release created: `v1.2.0`.
6. Release notes published.

For a MAJOR release, additionally:
7. Migration guide published.
8. RFC accepted documenting all breaking changes.
9. At least one MINOR release with all deprecations announced precedes
   the MAJOR.

### 5. Governance Model

#### PlayOS Foundation

The PlayOS Foundation is the legal steward of the project.  It holds
the trademark, domains, and cloud infrastructure.  It does **not**
control technical decisions — those are made by the maintainers.

#### Maintainer structure

| Role | Responsibility | Selection |
|---|---|---|
| **Spec Maintainer** | Approves RFCs, merges spec changes | Elected by existing Spec Maintainers |
| **API Maintainer** | Reviews and merges Platform API changes | Elected by API contributors |
| **Runtime Maintainer** | Reviews and merges Runtime changes | Elected by Runtime contributors |
| **Shell Maintainer** | Reviews and merges Shell changes | Elected by Shell contributors |
| **Cloud Maintainer** | Reviews and merges Cloud changes | Elected by Cloud contributors |

Maintainers serve 2-year terms.  There is no limit on terms.  A
maintainer may be removed by a 2/3 vote of all maintainers.

#### Decision making

| Decision type | Process | Approval |
|---|---|---|
| Platform model change | RFC | 2/3 of Spec Maintainers |
| API addition/change | RFC | Spec Maintainer + API Maintainer |
| Implementation detail | ADR | Relevant Maintainer |
| Bug fix, doc fix | PR | Any Maintainer |
| Deprecation | RFC | Spec Maintainer + API Maintainer |

#### RFC process (formalised)

1. **Proposal** — author creates `rfcs/NNNN-title.md` from template.
2. **Discussion** — PR opened; maintainers and community review.
3. **Decision** — after at least 7 days, maintainers vote.  Approved if
   2/3 of Spec Maintainers agree.
4. **Implementation** — if the RFC requires code changes, they land in
   the relevant repository after the RFC is merged.
5. **Spec update** — the RFC's content is incorporated into the PlayOS
   Book as part of the next Specification release.

RFCs may have status: `Draft`, `Accepted`, `Rejected`, `Superseded`,
`Deprecated`.

#### ADR process

1. **Decision made** — during implementation, a non-trivial choice is
   identified.
2. **ADR written** — author creates `adr/NNNN-title.md` from template.
3. **Review** — relevant Maintainer reviews and merges.
4. **Record** — the ADR is immutable once merged.  If the decision is
   later reversed, a new ADR is written that supersedes the old one.

ADR statuses: `Proposed`, `Accepted`, `Superseded`, `Deprecated`.

### 6. Contribution Model

#### Contributor License Agreement (CLA)

All contributors MUST sign a CLA before their first contribution is
merged.  The CLA grants the PlayOS Foundation a perpetual license to
use the contribution under the project's license (MIT/Apache-2.0 for
code, CC-BY-4.0 for specification).  The contributor retains copyright.

#### Contribution workflow

1. Fork the repository.
2. Create a feature branch.
3. Implement the change, including tests.
4. Open a pull request.
5. CI must pass (build, tests, lint).
6. At least one Maintainer must approve.
7. PR is merged by a Maintainer.

#### Code of Conduct

Contributors MUST follow the [Contributor Covenant](https://www.contributor-covenant.org/).
Violations are handled by the Foundation.

### 7. Commercial Ecosystem

PlayOS is an open platform, but commercial activity is encouraged:

| Activity | Policy |
|---|---|
| Selling apps on the marketplace | 70/30 revenue split (developer/platform); negotiable for OEM stores |
| Selling hardware preloaded with PlayOS | Free; must pass certification |
| Operating a private/enterprise store | Free; self-hosted |
| Forking the platform | Permitted under MIT/Apache-2.0; cannot use "PlayOS" trademark without certification |
| Building development tools | Permitted; no platform fee |
| Charging for cloud services | Permitted; must be transparent |

The PlayOS Foundation may charge a certification fee to cover testing
costs.  The certification fee is cost-recovery only — the Foundation is
a non-profit.

## Drawbacks

- Multi-repo versioning increases coordination overhead.  A change that
  spans Platform API + Runtime + Shell requires aligned releases.
- The two-year compatibility window may slow innovation compared to
  platforms that break compatibility more aggressively.
- Maintainer elections and formal governance add process overhead that
  may feel heavy for a small project.
- Independent versioning of 12+ artifacts creates complexity for
  developers ("which Runtime version do I need for API 1.2?").

## Rationale and Alternatives

- **Single monorepo with one version:** Simpler versioning but forces
  all components to release together.  Rejected — Cloud and Tools should
  be able to release independently of the Platform API.
- **No compatibility guarantees (break at will):** Faster iteration but
  unacceptable for a platform that asks developers to invest in building
  apps.  Rejected — the platform's value is the contract.
- **Linux-kernel-style "no breaking userspace":** Strongest guarantee
  but imposes permanent API surface.  PlayOS chooses a middle ground:
  compatibility within a MAJOR, breaking changes only at MAJOR
  boundaries with deprecation notice.
- **BDFL (Benevolent Dictator) governance:** Simpler for a small project
  but does not scale to a multi-stakeholder ecosystem.  Maintainer
  council is chosen for balanced decision-making.
- **No CLA:** Simpler contribution but creates legal risk for the
  Foundation (unclear license provenance).  CLA with retained copyright
  is the standard for open-source foundations (Apache, CNCF, Eclipse).

## Prior Art

- **Rust RFC process** (RFC-0000 template, FCP, team approval):
  directly inspires PlayOS's RFC process.
- **Kubernetes governance** (SIGs, maintainer elections, lazy
  consensus): model for multi-repo governance with independent
  maintainer groups.
- **Android API levels** (API level integer, `minSdkVersion`):
  reference for how a platform can version its API independently of the
  OS release.  PlayOS uses `MAJOR.MINOR` instead of integer levels.
- **Semantic Versioning** (semver.org): the base versioning scheme.
  PlayOS adopts semver with platform-specific rules for what constitutes
  MAJOR/MINOR/PATCH.
- **Vulkan API versioning** (API version + driver version independent):
  reference for how a platform API version can be independent of the
  implementation version.  PlayOS follows this pattern.

## Unresolved Questions

- What is the exact certification fee structure?  Cost-recovery implies
  it varies by device complexity.
- Should the Specification and Platform API always share the same MAJOR
  version?  Currently proposed as independent but aligned.
- How are initial maintainers selected before the project has enough
  contributors for elections?  Bootstrap: initial maintainers are the
  project founders; elections begin after 10 active contributors.
- Should the cloud services versioning be aligned with a single "PlayOS
  Cloud" version, or truly independent per service?

## Future Possibilities

- **LTS (Long-Term Support) releases:** Designate specific Runtime
  versions for extended support (3+ years) for enterprise and embedded
  deployments.
- **Compatibility test suite as a service:** Automated testing of apps
  against new Platform API versions before release.
- **Paid certification tiers:** "PlayOS Certified" vs. "PlayOS
  Compatible" (self-attested) with different trademark usage rights.
- **Federated governance:** OEM representatives on the maintainer
  council for hardware-specific decisions.
- **Plugin/extensions marketplace:** A separate governance model for
  third-party Platform API extensions.
