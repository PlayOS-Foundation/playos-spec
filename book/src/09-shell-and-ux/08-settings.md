# Settings

The **Settings** application provides system configuration. It is
accessible from the home screen and from Quick Settings.

## Layout

```
┌──────────────────────────────────────────┐
│  ← Settings                              │
├──────────────────────────────────────────┤
│                                          │
│  [👤 Account                 ▸]          │
│  [📶 Network                 ▸]          │
│  [🔊 Audio                   ▸]          │
│  [🖥  Display                 ▸]          │
│  [🎮 Controllers             ▸]          │
│  [🔋 Power & Battery         ▸]          │
│  [💾 Storage                 ▸]          │
│  [🔒 Privacy & Security      ▸]          │
│  [♿ Accessibility            ▸]          │
│  [🌐 Language & Region       ▸]          │
│  [🔄 System Updates          ▸]          │
│  [ℹ  About                   ▸]          │
│                                          │
└──────────────────────────────────────────┘
```

## Settings categories

| Category | Settings |
|---|---|
| **Account** | Sign in/out, profile, linked devices, friends |
| **Network** | Wi-Fi, Bluetooth, flight mode, proxy |
| **Audio** | Volume, mute, audio device, equalizer |
| **Display** | Brightness, resolution, night mode, HDR |
| **Controllers** | Pair, remap buttons, dead zones, vibration |
| **Power & Battery** | Sleep timer, battery saver, performance mode |
| **Storage** | Free space, manage applications, clear cache |
| **Privacy & Security** | Permissions, telemetry, crash reports |
| **Accessibility** | Screen reader, contrast, subtitles, button remapping |
| **Language & Region** | System language, date/time format, region |
| **System Updates** | Check for updates, update channel, last update |
| **About** | System version, device info, licenses |

## Navigation

- **D-pad / Left stick**: Scroll through settings list.
- **A**: Enter submenu.
- **B**: Go back.
- **Left/Right on toggle**: Toggle on/off.
- **Left/Right on slider**: Adjust value.

## Persistence

Settings are persisted to `/etc/playos/settings.toml` on the device
and (optionally) synced to the cloud for linked devices.

## OEM customization

OEMs can add settings under a "Device" category for hardware-specific
options (e.g., fan control, TDP settings). These settings use vendor
capabilities.

## Related

- [Quick Settings](09-quick-settings.md)
- [Accessibility](14-accessibility.md)
- [Device Profiles](../04-target-model/07-device-profiles.md)
