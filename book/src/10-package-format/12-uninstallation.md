# Uninstallation

Uninstallation removes the application bundle from the device. Save data
handling is a player choice.

## Process

1. **Confirm** — the shell prompts the player to confirm uninstallation.
2. **Save data prompt** — the player chooses whether to keep or delete
   save data.
3. **Remove bundle** — the runtime deletes the extracted package directory.
4. **Remove record** — the runtime removes the package from the local
   package database.
5. **Notify shell** — the shell removes the application from the library
   and game switcher.

## Save data handling

Save data lives outside the package directory (in `Storage::SavePath()`).
During uninstallation, the player is asked:

> "Keep save data? You can delete it later from Settings."

| Choice | Result |
|---|---|
| **Keep** | Save data remains on disk. If the application is reinstalled, saves are available. |
| **Delete** | Save data and configuration are deleted permanently. |

## Cloud saves

If cloud saves are enabled, uninstalling the application does not delete
cloud saves. The player may manage cloud saves through the cloud settings
or developer portal.

## Requirements

- Uninstallation MUST require player confirmation.
- Save data MUST NOT be deleted without explicit player choice.
- The runtime MUST clean up all extracted package files — no orphaned
  directories or files.
- Cloud saves MUST survive uninstallation.

## Related

- [Installation](11-installation.md)
- [Storage API](../06-platform-api/10-storage-api.md)
- [Cloud Saves](../11-cloud-and-marketplace/11-cloud-saves.md)
