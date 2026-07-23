# System and Application Updates

The Update Service manages **over-the-air (OTA) updates** for the
PlayOS Runtime and installed applications.

## Update types

| Type | What updates | Frequency | User-visible |
|---|---|---|---|
| **Runtime update** | Platform API, Runtime services, Shell | Monthly / as needed | Reboot required |
| **Application update** | Installed .gpk packages | Per developer schedule | Restart of that app |
| **Security patch** | Critical fixes | Immediately | May require reboot |
| **Device profile update** | Input mappings, capability declarations | As needed | None (hot-reload) |

## Runtime update process

1. Update Service checks for available updates (configurable server).
2. Downloads the update package (signed .gpk or system image).
3. Verifies the signature against the PlayOS Foundation's public key.
4. Pre-installs the update in a spare partition/slot (A/B update).
5. Prompts the user to reboot.
6. On reboot, the bootloader switches to the updated slot.

If the update fails, the bootloader falls back to the previous slot
(atomic rollback).

## Application update process

1. Update Service queries the application's configured update server
   (manifest `updateURL`).
2. Downloads the new .gpk.
3. Verifies the package signature (must match the installed
   application's signing key).
4. Installs the update alongside the existing installation.
5. Prompts the user or applies automatically based on preference.
6. Restarts the application.

## Update channels

Applications can use [release channels](../10-package-format/14-release-channels.md)
(stable, beta, nightly). The Update Service respects the user's
channel preference.

## Self-hosted updates

The Update Service supports pointing at a self-hosted update server.
The base URL is configurable:

- Runtime updates: PlayOS Cloud or self-hosted
- Application updates: developer's server (configured in manifest)

## Rollback

- **Runtime**: A/B partition scheme. Failed updates roll back
  automatically.
- **Applications**: The previous version is retained until the new
  version is confirmed stable (runs for 5 minutes without crashing).

## Related

- [Package Format — Updates and Patches](../10-package-format/13-updates-and-patches.md)
- [Release Channels](../10-package-format/14-release-channels.md)
- [Signing](../10-package-format/10-signing.md)
- [Self-Hostable by Design](../02-platform-principles/06-self-hostable-by-design.md)
