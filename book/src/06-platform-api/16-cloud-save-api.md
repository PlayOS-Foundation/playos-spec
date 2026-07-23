# Cloud Save API

The Cloud Save API synchronizes save data across devices through PlayOS
Cloud. It is a **runtime-only** feature gated on the optional capability
`cloud.saves` (enum: `Capability::CloudSave`).

## Purpose

- **Cross-device sync** — a player's save data follows them across devices.
- **Backup** — save data is stored in the cloud, recoverable if the device
  is lost or reset.
- **Conflict resolution** — the platform handles conflicts when the same
  save is modified on two devices.

## API concepts

The Cloud Save API is designed to be minimal from the application side:

```cpp
namespace PlayOS {
namespace CloudSave {

// Synchronize save data with the cloud.
// Blocks until sync completes or fails.
SyncResult Sync();

// Check whether cloud saves are enabled for this application.
bool IsEnabled();

} // namespace CloudSave
} // namespace PlayOS
```

The application does not specify files or paths — the platform syncs
everything under `Storage::SavePath()`. This keeps the API simple and
prevents applications from accidentally leaking saved state.

## Sync model

1. Application calls `Sync()` — typically during a loading screen, on
   startup, or on shutdown.
2. The platform compresses save data, uploads to object storage, and
   updates metadata.
3. On another device, `Sync()` downloads the latest save and applies it,
   resolving conflicts with a last-write-wins or user-prompt strategy.

## Conflict resolution

When the same save has been modified on two devices since the last sync:

- The platform MAY resolve with last-write-wins.
- The platform MAY present a conflict-resolution UI via the overlay.
- The strategy is defined by the cloud service; the application is not
  involved.

## Capability

| Capability | Enum | Status |
|---|---|---|
| `cloud.saves` | `Capability::CloudSave` | Optional (runtime-only) |

## Failure behavior

When `Capability::CloudSave` is absent:
- `IsEnabled()` returns `false`.
- `Sync()` returns `SyncResult::NotAvailable`.

When the cloud service is unreachable:
- `Sync()` returns `SyncResult::NetworkError` or similar.
- Local save data is never modified by a failed sync.

## Requirements

- `Sync()` MUST NOT modify local save data until the cloud operation
  completes successfully.
- The platform MUST sync the entire save directory — applications must not
  need to enumerate files.
- The cloud service MUST encrypt save data at rest.
- Save data size SHOULD be reasonable (recommended: under 100 MB per title).

## Related

- [Storage API](10-storage-api.md)
- [Cloud Architecture](../11-cloud-and-marketplace/03-cloud-architecture.md)
- [Cloud Saves](../11-cloud-and-marketplace/11-cloud-saves.md)
- RFC-0004 (Platform API Surface)
- RFC-0007 (Cloud and Marketplace Architecture)
