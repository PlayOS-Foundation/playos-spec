# Storage Porting

The storage backend provides filesystem paths for persistent save
data, configuration, and cache directories.

## Interface to implement

```cpp
// backends/storage_backend.h
class IStorageBackend {
public:
    virtual const char* SavePath() = 0;    // Persistent save data
    virtual const char* ConfigPath() = 0;  // Configuration data
};
```

The backend **must** ensure the returned directories exist (creating
them if needed). Paths are absolute and platform-appropriate.

### What goes where

| Path | Purpose | Cleared on uninstall |
|---|---|---|
| `SavePath()` | Game saves, user progress | Usually kept |
| `ConfigPath()` | Settings, preferences | Usually kept |

Future extensions may add `CachePath()` and `LogsPath()`.

## Linux implementation (XDG)

Uses the [XDG Base Directory Specification]:

- `SavePath()` → `$XDG_DATA_HOME/PlayOS/saves/`
  (defaults to `~/.local/share/PlayOS/saves/`)
- `ConfigPath()` → `$XDG_DATA_HOME/PlayOS/config/`
  (defaults to `~/.local/share/PlayOS/config/`)

Directories are created with `0755` permissions on first access.

## Windows implementation

Uses the Windows known-folder API:

- `SavePath()` → `%LOCALAPPDATA%/PlayOS/saves/`
- `ConfigPath()` → `%LOCALAPPDATA%/PlayOS/config/`

On Windows, `%LOCALAPPDATA%` is the correct location for
machine-local application data (as opposed to `%APPDATA%` which
roams with the user profile in enterprise environments). PlayOS data
is machine-local.

## Null backend

Returns `"./PlayOS/saves"` and `"./PlayOS/config"` relative to the
current working directory. Suitable for SDK Targets during
development.

## Quota and sandboxing

The storage backend is not responsible for quota enforcement.
Storage quotas are enforced by the [Runtime Architecture](
../08-runtime-architecture/01-overview.md) process sandbox — each
game runs as a separate user/namespace with a fixed storage budget.
The backend only provides the path; the runtime enforces the limits.

## Porting steps

1. Create `src/backends/<platform>/<platform>_storage_backend.cpp`.
2. Pick platform-appropriate base directories.
3. Implement `SavePath()` and `ConfigPath()` with directory creation.
4. Call `Capabilities::RegisterCapability(Capability::StorageLocal)`.
5. Register in CMake.

## Implementation gaps

| Platform | Backend status | Notes |
|---|---|---|
| Linux | ✅ Complete | XDG Base Directory |
| Windows | ✅ Complete | `SHGetFolderPath` / `%LOCALAPPDATA%` |
| Android | ❌ Missing | Need `Context.getFilesDir()` via JNI |
| PS Vita | ❌ Missing | Need `ux0:data/` path |

## Related

- [Backend Porting Model](03-backend-porting-model.md)
- [Package Format](../10-package-format/01-overview.md) — where game data lives
- [Runtime Architecture](../08-runtime-architecture/01-overview.md) — quota enforcement
