# What Is a Capability?

A **capability** is a named, discoverable feature that a PlayOS target may
or may not provide. It is the unit of feature detection in the PlayOS
platform.

## Anatomy of a capability

Every capability has:

| Property | Description |
|---|---|
| **Identifier** | A stable, namespaced string, e.g. `input.touch`, `cloud.saves`, `power.battery` |
| **Group** | The top-level namespace: `input`, `display`, `storage`, `audio`, `power`, `network`, `cloud`, `system`, `vendor` |
| **Status** | One of `required`, `optional`, or `vendor` |
| **Description** | Human-readable explanation of what the capability provides |
| **Feature level** (optional) | Graded support when a capability is not simply present/absent, e.g. touch single-touch vs multi-touch |

## Capability vs platform check

A capability query asks "what can this device do?" A platform check asks
"what operating system is this?" The former adapts; the latter hardcodes.

```cpp
// Platform check — fragile, not future-proof
#ifdef __ANDROID__
    // assume touch
#endif

// Capability query — adaptive, future-proof
if (PlayOS::Capabilities::Has(PlayOS::Capability::Touch)) {
    // touch is available
}
```

The capability approach means the same binary can run on a handheld, a
desktop, and a set-top box without recompilation.

## Capability vs feature flag

A **feature flag** is a build-time decision: compile with or without a
feature. A **capability** is a runtime property: the binary always
includes the code path, and decides at runtime whether to use it.

This is essential because two devices running the same PlayOS binary
may differ in hardware — a handheld with touch vs a desktop without.
Feature flags cannot express that difference.

## Where capabilities live

Capabilities are registered in the
[Capability Registry](../../schemas/capability-registry.schema.json). A
device declares which capabilities it provides in its
[Device Profile](../12-device-model-and-porting/02-device-profile-schema.md).
The backend implements the corresponding Platform API surface, and the
Capabilities module reports them to the application at runtime (see
[Runtime Capability Discovery](05-runtime-capability-discovery.md)).

## Related

- [Required Capabilities](03-required-capabilities.md)
- [Optional Capabilities](04-optional-capabilities.md)
- [Capability Groups](07-capability-groups.md)
- RFC-0003 (Capability Model)
