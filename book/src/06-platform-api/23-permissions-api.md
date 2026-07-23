# Permissions API

The Permissions API lets applications check and request permissions at
runtime. Permissions are declared in the package manifest at build time
and granted by the player at install time. The runtime API lets the
application verify state and request re-granting if needed.

## Purpose

- **Check permission state** — is a permission currently granted?
- **Request permission** — ask the player to grant a permission (triggers
  system UI).
- **Permission lifecycle** — permissions can be granted, denied, or revoked
  at any time by the player through system settings.

## API concepts

```cpp
namespace PlayOS {
namespace Permissions {

enum class PermissionState {
    Granted,     // Permission is active
    Denied,      // Player denied (temporarily)
    Revoked,     // Player revoked through settings (permanent denial)
    NotRequested // Never asked
};

// Check the current state of a permission.
PermissionState GetState(const std::string& permissionId);

// Request a permission. May trigger system UI. Returns the new state.
PermissionState Request(const std::string& permissionId);

} // namespace Permissions
} // namespace PlayOS
```

## Permission identifiers

Permissions use the same lowercase dot-separated format as capabilities:

| Permission | Description |
|---|---|
| `storage.local` | Read/write save and config data |
| `storage.external` | Access external storage (SD card, USB) |
| `network.internet` | Internet access |
| `network.local` | Local network discovery |
| `audio.microphone` | Microphone access |
| `system.overlay` | Display system overlay |

The full registry is maintained in the
[Permission Registry](../../schemas/permission-registry.schema.json).

## Manifest declaration

Permissions must be declared in the package manifest (`manifest.json`):

```json
{
  "permissions": ["storage.local", "network.internet"]
}
```

Undeclared permissions cannot be requested at runtime — `Request()` returns
`Denied` without prompting the player.

## Player consent

At install time, the runtime presents the declared permission list to the
player. The player may grant all, deny some, or deny all. After install,
the player may change permissions at any time through system settings.

## Capability

Permissions are not gated by a capability — the Permissions API is available
on all conformant targets. On SDK Targets without a system permission UI,
`Request()` returns `Granted` for declared permissions automatically.

## Requirements

- `Request()` MUST NOT block indefinitely — it returns after the player
  responds or after a timeout.
- Permissions revoked through system settings MUST be reflected in
  `GetState()` immediately.
- Undeclared permissions MUST NOT be grantable.
- The permission registry MUST define risk levels (low, medium, high) for
  each permission to guide the player-facing UI.

## Related

- [Package Format](../10-package-format/01-overview.md) — manifest
  declaration
- [Permissions](../13-security-privacy-and-trust/04-permissions.md) —
  security model
- `schemas/permission-registry.schema.json`
