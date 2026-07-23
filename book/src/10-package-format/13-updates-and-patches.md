# Updates and Patches

Updates replace an installed application with a newer version while
preserving save data, configuration, and entitlements.

## Update flow

1. **Check** — the runtime periodically checks the marketplace (or
   configured store sources) for updates to installed packages.
2. **Download** — if an update is available, the runtime downloads the
   new `.gpk` in the background.
3. **Notify** — the shell shows an update available notification.
4. **Install** — the player initiates the update. The runtime:
   - Verifies the signature of the new package.
   - Preserves save data and configuration (they live outside the package
     directory).
   - Replaces the extracted bundle with the new version.
   - Updates the version in the local package database.
5. **Complete** — the application is ready to launch with the new version.

## Save data preservation

Save data and configuration are stored outside the package directory
(see [Storage API](../06-platform-api/10-storage-api.md)). Updates replace
only the package bundle — save data is never touched.

Applications MUST maintain backward compatibility with save data from
the previous MAJOR version. Breaking save data format changes require a
new package `id` (a separate product).

## Delta updates

Binary patch updates (downloading only changed bytes) are **deferred**.
The current model downloads the full `.gpk` for every update.

## Automatic updates

The player configures update behavior in Settings:

| Setting | Behavior |
|---|---|
| **Auto-update** | Download and install updates automatically in the background. |
| **Notify** | Download updates, notify player, wait for confirmation. |
| **Manual** | Check for updates, download and install only when the player requests. |

## Rollback

The runtime does not provide automatic rollback. If an update causes
issues, the player must manually reinstall the previous version (if
available from the store or a backup).

## Requirements

- Updates MUST preserve save data and configuration.
- The runtime MUST verify the new package's signature before replacing
  the existing installation.
- Updates MUST be atomic — a failed update leaves the existing
  installation intact.

## Related

- [Installation](11-installation.md)
- [Storage API](../06-platform-api/10-storage-api.md)
- [Updates and CDN](../11-cloud-and-marketplace/16-updates-and-cdn.md)
