# Cloud Saves

**Cloud Saves** synchronize application save data across devices. A
player can start on one device and continue on another without
manually transferring files.

## How it works

1. The application writes save data to the Platform API's
   `Storage::SaveFile()`.
2. The Cloud Sync service detects the change and uploads it to the
   cloud.
3. On another device, the service downloads the updated save.
4. The application reads the save via `Storage::LoadFile()`.

The application does not need to know that cloud saves exist. It just
uses the Storage API. The Runtime handles sync transparently.

## Conflict resolution

When the same save is modified on two devices before sync:

1. The Cloud Sync service detects the conflict.
2. It stores both versions and marks the save as conflicted.
3. The application receives a `SaveConflict` callback:

```cpp
void OnSaveConflict(const Storage::ConflictInfo& info) {
    // info.localVersion — the version on this device
    // info.remoteVersion — the version from the cloud
    // info.baseVersion — the last known common version
    
    // Application resolves the conflict
    Storage::ResolveConflict(info.saveId, Storage::Resolution::UseLocal);
}
```

The application is responsible for resolving the conflict. The Runtime
provides both versions and the common base.

## Save slots

Each application has multiple save slots (default: 10). Slots are
identified by an integer index. The application chooses which slot to
save to:

```cpp
Storage::SaveToSlot(0, saveData, saveSize);
Storage::LoadFromSlot(2, &saveData, &saveSize);
```

## Storage quota

| Tier | Quota per application |
|---|---|
| Free | 100 MB |
| PlayOS Plus (future) | 1 GB |

## Self-hosting

The Cloud Sync service uses an S3-compatible backend. Self-hosted
instances can use MinIO, Ceph, or any S3-compatible storage.

## Capability

Cloud saves are gated by `cloud.saves`. The application syncs saves
only when this capability is present.

## Related

- [Cloud Save API](../06-platform-api/16-cloud-save-api.md)
- [Storage API](../06-platform-api/06-storage-api.md)
- [Cloud Architecture](03-cloud-architecture.md)
- [Self-Hosting Principles](02-self-hosting-principles.md)
