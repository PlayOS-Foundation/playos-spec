---
rfc: 0001
title: Platform Principles
status: Accepted
authors: []
created: 2026-01-01
---

# RFC-0001: Platform Principles

> **Status:** Accepted

## Summary

This RFC defines the founding principles of PlayOS. These principles are the
constitution of the platform: every other RFC, specification chapter, and
implementation decision must be consistent with them. The principles are:
specification-first, engine-agnostic, console-first, capability-based,
runtime-independent, self-hostable by design, open platform, long-term
compatibility, and security and trust.

## Motivation

PlayOS spans many repositories, devices, engines, runtimes, and contributors.
Without a shared set of principles, each implementation would make locally
reasonable choices that collectively fragment the platform. A small, explicit
set of principles lets humans and AI agents make consistent decisions without
central coordination, and gives developers and hardware vendors confidence
that the platform will remain coherent and stable over time.

## Guide-Level Explanation

PlayOS is governed by nine principles.

1. **Specification First.** Everything is specified before it is implemented.
   The PlayOS Book is the source of truth; implementations follow it.
2. **Engine Agnostic.** The Platform API never depends on a single game
   engine. Raylib is the reference engine only.
3. **Console First.** The experience is controller-first and touch-aware;
   keyboard and mouse are optional.
4. **Capability Based.** Applications query capabilities instead of checking
   platforms or operating systems.
5. **Runtime Independent.** The Platform API is usable on full runtime
   devices and on SDK targets that do not run the PlayOS runtime.
6. **Self-Hostable by Design.** Every cloud or marketplace feature has a
   self-hostable implementation; no forced dependence on a central provider.
7. **Open Platform.** The specification, the Platform API, and the reference
   implementations are open. Anyone may implement the specification.
8. **Long-Term Compatibility.** Once the Platform API reaches a stable
   release, applications continue to build and run on future compatible
   implementations wherever practical.
9. **Security and Trust.** Packages are signed, permissions are explicit, and
   the platform is designed to protect players and developers.

## Reference-Level Explanation

The principles are normative in the following sense:

- **Specification First** means a change that affects platform behavior MUST
  be reflected in `book/`, and, when non-trivial, in an RFC, before it is
  implemented in another repository.
- **Engine Agnostic** means public Platform API headers MUST NOT include
  engine headers, and no API may require a specific engine to function.
- **Console First** means reference UX and input models assume a controller
  as the primary device; other input methods are additive.
- **Capability Based** means feature availability MUST be discoverable at
  runtime through the capability API (see RFC-0003). Platform/OS checks in
  application code are disallowed where a capability query is possible.
- **Runtime Independent** means the Platform API surface MUST separate the
  portable subset (available to SDK targets) from runtime-only features
  (see RFC-0002).
- **Self-Hostable by Design** means any hosted service defined by PlayOS MUST
  have a documented self-hosted deployment.
- **Open Platform** means the specification is published under an open
  license and no conformant implementation requires permission to exist.
- **Long-Term Compatibility** means breaking changes to a stable Platform API
  MUST follow the versioning and deprecation policies (governance part).
- **Security and Trust** means package signing and a permission model are
  mandatory parts of the platform, not optional add-ons.

Each principle has a corresponding chapter in
`book/src/02-platform-principles/`, which is the authoritative prose
definition. This RFC records the decision to adopt them as a set.

## Drawbacks

- Principles constrain design freedom; some locally convenient shortcuts
  (for example, an `#ifdef` platform check) are disallowed.
- Specification-first adds up-front process cost before code can be written.
- Requiring self-hostable implementations increases the engineering surface
  of cloud and marketplace features.

These costs are accepted deliberately in exchange for long-term coherence.

## Rationale and Alternatives

- **No explicit principles.** Rejected: leads to drift across repositories
  and contributors.
- **Implementation-first (build then document).** Rejected: the platform's
  value is the contract; letting implementations define behavior undermines
  portability and compatibility.
- **Fewer principles.** Considered, but each principle addresses a distinct
  failure mode observed in comparable platforms (engine lock-in, platform
  checks, central-provider lock-in, compatibility breakage).

The chosen set mirrors how durable platform standards (POSIX, Vulkan,
OpenXR) separate a stable contract from its implementations.

## Unresolved Questions

- The precise normative wording (MUST/SHOULD/MAY) for each principle is
  finalized in the corresponding book chapters.
- The exact stability guarantees under **Long-Term Compatibility** depend on
  the versioning policy, which is specified separately in the governance
  part.
