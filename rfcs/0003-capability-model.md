---
rfc: 0003
title: Capability Model
status: Accepted
authors: []
created: 2026-01-01
---

# RFC-0003: Capability Model

> **Status:** Accepted

## Summary

PlayOS applications discover what a device can do by querying **capabilities**
at runtime, rather than checking the platform or operating system. A
capability is a named, discoverable feature (for example `input.touch` or
`cloud.saves`). This RFC defines what a capability is, how capabilities are
categorized, how they are discovered, and how the platform behaves when a
capability is absent.

## Motivation

Devices differ widely: a handheld may have touch and battery; a desktop may
have neither but support cloud saves; an SDK target may lack the runtime
entirely. Hardcoded platform checks (`#ifdef ANDROID`) bake in assumptions
that break whenever a new device appears, and they make one binary unable to
adapt across devices. A capability model makes differences explicit,
discoverable, and future-proof: new devices are added by declaring
capabilities in a profile, not by changing application code or the SDK.

## Guide-Level Explanation

Instead of:

```cpp
#ifdef ANDROID
    // touch code
#endif
```

applications write:

```cpp
if (PlayOS::Capabilities::Has(PlayOS::Capability::Touch)) {
    auto touches = PlayOS::Input::TouchPoints();
}
```

Capabilities are grouped:

- **Required** — every conformant target provides them (for example basic
  input, storage, display information, lifecycle).
- **Optional** — may or may not be present (for example touch, battery,
  cloud saves, overlay, marketplace, suspend/resume).
- **Vendor** — device- or vendor-specific extensions (for example fan or TDP
  control on certified hardware).

Because runtime-only features from RFC-0002 are expressed as capabilities,
the same application runs on both SDK targets and runtime devices, using more
features where they are available.

## Reference-Level Explanation

The capability model specifies:

- **Identity.** Each capability has a stable string identifier in a
  namespaced form, for example `input.touch`, `cloud.saves`,
  `system.overlay`, `power.battery`. Identifiers are recorded in the
  capability registry (`schemas/capability-registry.schema.json` and the
  appendix).
- **Discovery.** The Platform API exposes at least:
  - `PlayOS::Capabilities::Has(capability)` — is a capability present,
  - `PlayOS::Capabilities::List()` — enumerate present capabilities.
- **Declaration.** A device declares its capabilities through its device
  profile; a backend implements the corresponding Platform API surface.
- **Categories.** Each registered capability has a status of `required`,
  `optional`, or `vendor`, and belongs to a capability group.
- **Failure behavior.** Calling an API guarded by an absent capability MUST
  fail predictably (a well-defined error or documented no-op), never crash.
  Applications are expected to guard optional capabilities with a query.
- **Required-capability guarantee.** A target that fails to provide a
  required capability is non-conformant and MUST NOT claim the corresponding
  certification level.

This model underpins the device model (profiles + backends), the target
model (RFC-0002), and certification (conformance is defined partly in terms
of required capabilities).

## Drawbacks

- Every feature must be assigned a capability and registered, adding process
  overhead.
- Applications must remember to query optional capabilities; forgetting leads
  to relying on features that may be absent. Tooling and lints can help.
- A large capability registry must be governed to avoid duplication or drift.

## Rationale and Alternatives

- **Platform/OS checks (`#ifdef`).** Rejected: not future-proof, not
  discoverable at runtime, and prevents a single adaptive binary.
- **Feature flags per build.** Rejected: shifts the problem to build
  configuration and still cannot adapt at runtime across devices.
- **Version-number gating.** Rejected: coarse and does not express that two
  devices at the same version can differ in hardware features.

Capability discovery mirrors how web and graphics platforms expose optional
features (feature detection, extension queries) and is the natural fit for a
specification-first, multi-device platform.

## Unresolved Questions

- The initial contents of the required/optional/vendor capability registry
  are enumerated in the capability-model chapters and the appendix.
- Whether capabilities need **feature levels** (graded support rather than
  boolean presence) is tracked in the capability-model part and may be
  refined in a follow-up RFC.
