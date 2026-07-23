# Device Profiles

A **device profile** is a TOML file that declares what a device is and
what it can do. The runtime reads it at startup to configure backends and
populate the capability registry.

## Purpose

A device profile answers:

- What is this device? (identity, type)
- What input does it have? (gamepad, touchscreen, special buttons)
- What display does it have? (resolution, refresh rate, safe area)
- What capabilities does it provide? (required, optional, vendor)
- What is the physical-to-logical button mapping?

## Format

Device profiles are TOML files following
[`device-profile.schema.json`](../../schemas/device-profile.schema.json):

```toml
[device]
id = "rog-ally"
name = "ASUS ROG Ally"
targetType = "runtime-device"

[input]
homeButton = "asus_armoury"
quickSettingsButton = "asus_command_center"
builtInControls = "gamepad0"
touchscreen = "touch0"

[display]
defaultWidth = 1920
defaultHeight = 1080
refreshRate = 120

[display.safeAreaInsets]
# No notch or rounded corners on the Ally.

[capabilities]
required = ["input.basic", "storage.local", "display.info", "lifecycle.basic"]
optional = ["input.touch", "power.battery", "display.brightness", "network.info"]
vendor = ["vendor.rog-ally.fan-control", "vendor.rog-ally.tdp-control"]
```

## Location

On Runtime Devices: `/etc/playos/device-profiles/<id>.toml`

In the source tree:
`playos-reference-devices/<device>/device-profile.toml`

## Loading

The `DeviceProfile` class in `playos/device_profile.h` loads and parses
device profiles. At startup:

1. The runtime reads the device profile for the current hardware.
2. Extracts input mappings, display properties, and capabilities.
3. Configures backends with the extracted values.
4. Registers capabilities with the capability registry.

## Requirements

- Every Runtime Device MUST ship a device profile.
- SDK Targets MAY use a minimal profile or the `DeviceProfile::Default()`.
- Profiles MUST validate against the schema.
- Profiles MUST accurately reflect the hardware — no false capability
  claims.

## Related

- [Device Profile Schema](../12-device-model-and-porting/02-device-profile-schema.md)
- `schemas/device-profile.schema.json`
- `playos/device_profile.h` (Platform API header)
- RFC-0006 (Device Profile Format)
