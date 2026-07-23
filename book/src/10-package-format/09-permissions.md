# Permissions

Permissions control what an application may access — storage, network,
microphone, and other system resources. Permissions are declared in the
manifest and granted by the player at install time.

## Declaration

Permissions are listed in `manifest.json`:

```json
{
  "permissions": ["storage.local", "network.internet"]
}
```

The full permission registry is defined in
[`permission-registry.schema.json`](../../schemas/permission-registry.schema.json).

## Permission lifecycle

1. **Declare** — developer lists permissions in the manifest.
2. **Present** — at install time, the runtime shows the permission list
   to the player.
3. **Consent** — the player grants or denies each permission.
4. **Enforce** — the runtime enforces granted permissions at the API level.
5. **Revoke** — the player may revoke permissions at any time through
   system settings.

## Permission categories

| Permission | Risk | Description |
|---|---|---|
| `storage.local` | Low | Read/write save and config data |
| `storage.external` | Medium | Access external storage (SD card, USB) |
| `network.internet` | Low | Internet access |
| `network.local` | Medium | Local network discovery and communication |
| `audio.microphone` | High | Microphone access |
| `system.overlay` | Medium | Display system overlay |

## Runtime API

Applications check permission state at runtime via the
[Permissions API](../06-platform-api/23-permissions-api.md):

```cpp
auto state = PlayOS::Permissions::GetState("storage.local");
if (state == PlayOS::Permissions::PermissionState::Granted) {
    // safe to use storage
}
```

## Requirements

- Undeclared permissions MUST NOT be grantable at runtime.
- Revoked permissions MUST be reflected immediately in the Permissions API.
- The player MUST be able to review and change permissions at any time
  through system settings.

## Related

- [Permissions API](../06-platform-api/23-permissions-api.md)
- [Security, Privacy and Trust](../13-security-privacy-and-trust/01-overview.md)
- `schemas/permission-registry.schema.json`
