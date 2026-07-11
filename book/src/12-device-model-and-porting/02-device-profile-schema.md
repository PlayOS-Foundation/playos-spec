# Device Profile Schema

A **device profile** is a TOML file that describes a PlayOS hardware target.
Adding a new device means writing a profile and, where needed, a backend for
missing capabilities — not recompiling the platform.

The canonical specification is **RFC-0006** (`rfcs/0006-device-profile-format.md`).
The JSON Schema is at `schemas/device-profile.schema.json`.

## Why TOML?

Profiles are human-authored configuration. TOML supports comments, is readable
without tooling, and maps cleanly to key-value structures. JSON was rejected
for this use case (no comments, harder to hand-edit).

## Sections

| Section | Required | Purpose |
|---|---|---|
| `[device]` | ✅ | Identity: `id`, `name`, `targetType` |
| `[input]` | Runtime devices | Maps vendor buttons → logical PlayOS inputs |
| `[display]` | ✅ | Native resolution, refresh rate, safe area insets |
| `[capabilities]` | ✅ | Boolean map of capability IDs → present/absent |
| `[power]` | Optional | sysfs paths for TDP, fan, brightness control |
| `[vendor]` | Optional | Vendor-specific extensions (namespaced) |

## Full Example (ROG Ally)

```toml
[device]
id = "rog-ally"
name = "ASUS ROG Ally"
targetType = "runtime-device"

[input]
home_button = "asus_armoury"
quick_settings_button = "asus_command_center"
built_in_controls = "gamepad0"
touchscreen = "touch0"

[display]
default_width = 1920
default_height = 1080
refresh_rate = 120

[capabilities]
"input.basic" = true
"input.touch" = true
"display.info" = true
"display.brightness" = true
"power.battery" = true
"system.overlay" = true
```

## Input Mapping Vocabulary

The `[input]` section uses symbolic names. These are resolved to platform key
codes at runtime by `InputMapping` in `playos-platform-api`.

See RFC-0006 Appendix for the full vocabulary. Common names:

| Name | Description | Device |
|---|---|---|
| `xbox_guide` | Xbox/PlayStation guide button | Generic controllers |
| `asus_armoury` | Armoury Crate button (left) | ROG Ally |
| `asus_command_center` | Command Center button (right) | ROG Ally |
| `steamdeck_qam` | Quick Access Menu (…) | Steam Deck |
| `keyboard_f1` | F1 key as Home fallback | Desktops, VMs |

## Profile Locations

- **Source tree:** `playos-reference-devices/<device>/device-profile.toml`
- **Runtime:** `/etc/playos/device-profiles/<id>.toml`
- **Kernel cmdline override:** `playos.profile=<id>` selects profile at boot

## Profile ↔ Capability Model

The `[capabilities]` section feeds `PlayOS::Capabilities::Has()`. A capability
declared `true` means the hardware supports it. If no backend implements it,
the API returns a graceful no-op — the profile is the source of truth, not the
backend.

## Profile ↔ Input Backend

The Linux evdev backend reads `[input].home_button` at startup. It resolves
the symbolic name through the `InputMapping` lookup table to one or more evdev
key codes. If no profile is present, the backend falls back to hardcoded
defaults (`BTN_MODE`, common vendor codes).

## Schema Validation

The JSON Schema at `schemas/device-profile.schema.json` defines the canonical
structure. Tools can validate profiles against it:

```sh
# Using a JSON Schema validator (ajv, check-jsonschema, etc.)
check-jsonschema --schemafile schemas/device-profile.schema.json \
    playos-reference-devices/rog-ally/device-profile.toml
```

## See Also

- [RFC-0006: Device Profile Format](https://github.com/PlayOS-Foundation/playos-spec/blob/main/rfcs/0006-device-profile-format.md)
- [ROG Ally Reference](12-rog-ally-reference.md)
- [Input Routing](../08-runtime-architecture/10-input-routing.md)
- [Capability Model](../05-capability-model/01-overview.md)
