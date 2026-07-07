---
rfc: 0006
title: Device Profile Format
status: Accepted
authors: []
created: 2026-01-01
---

# RFC-0006: Device Profile Format

> **Status:** Accepted

## Summary

Every PlayOS device is described by a **device profile** — a TOML file that
declares identity, input mappings, display characteristics, capabilities,
and vendor extensions. Profiles are how a new hardware target is added
without changing the SDK. Schema:
[`schemas/device-profile.schema.json`](https://github.com/PlayOS-Foundation/playos-spec/blob/main/schemas/device-profile.schema.json).

## Motivation

Without profiles, hardware differences would be hard-coded (`#ifdef ROG_ALLY`)
or buried in per-platform backends. Profiles make the differences declarative:
adding a device means writing a TOML file and, where needed, a backend for
missing capabilities. The same binary adapts to different devices by reading
the profile through the capability and input APIs.

## Guide-Level Explanation

A device profile answers four questions:

1. **What is this device?** (`id`, `name`, `targetType`)
2. **How are physical inputs mapped?** (which hardware button is `Home`)
3. **What does the display look like?** (native res, refresh rate)
4. **What can this device do?** (capabilities: battery, touch, overlay, etc.)

Example excerpt (ROG Ally):

```toml
[device]
id = "rog-ally"
name = "ASUS ROG Ally"
targetType = "runtime-device"

[input]
home_button = "asus_armoury"
quick_settings_button = "asus_command_center"

[display]
default_width = 1920
default_height = 1080
refresh_rate = 120

[capabilities]
"input.basic" = true
"input.touch" = true
"power.battery" = true
```

## Reference-Level Explanation

### Format
Profiles are **TOML** (human-readable, comment-supported). The JSON Schema
in `schemas/device-profile.schema.json` is the canonical structure definition.

### Sections

| Section | Required | Purpose |
|---|---|---|
| `[device]` | yes | `id`, `name`, `targetType` |
| `[input]` | runtime devices | Maps vendor buttons to logical PlayOS inputs |
| `[display]` | yes | Native resolution, refresh rate |
| `[capabilities]` | yes | Boolean map of capability IDs → presence |
| `[power]` | optional | TDP, fan, brightness paths (certified hardware) |
| `[vendor]` | optional | Vendor-specific extensions (namespaced) |

### Relationship to capabilities

The `[capabilities]` section feeds the capability registry. `Has()` returns
true for capabilities declared `true`. Required capabilities MUST be `true`.

### Where profiles live
- Runtime devices: `/etc/playos/device-profiles/<id>.toml`
- Source tree: `playos-reference-devices/<device>/device-profile.toml`

### Profile ↔ backend
A profile declares a capability; the backend implements it. If a capability
is `true` but no backend is present, the API returns a graceful empty/no-op
response (no crash). The profile is the source of truth.

## Drawbacks

- TOML is less tooled for validation than JSON, but far more readable for
  human-authored config.
- Profiles alone cannot express complex hardware quirks (e.g. "this button
  needs a 200ms debounce") — those remain in backends.
- Profile and backend must stay in sync; a capability declared true but
  unimplemented is a silent no-op.

## Rationale and Alternatives

- **JSON:** Rejected for human-authored config — lacks comments, less
  readable.
- **Compiled-in C++ structs:** Rejected — requires SDK recompilation to
  add a device.
- **YAML:** Considered; TOML preferred for simpler key-value mapping.

## Unresolved Questions

- Whether profiles should be independently versioned.
- Conditional capabilities (e.g. "brightness only on AC") — out of scope
  for now.
