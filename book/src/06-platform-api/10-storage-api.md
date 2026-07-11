# Storage API

The Storage API gives applications a stable, platform-provided location to
write persistent data.  Applications MUST NOT hard-code paths or use
operating-system calls like `getenv("HOME")` directly — doing so breaks
portability across runtime devices, SDK targets, and future container models.

## Capability requirement

The Storage API is gated on the `storage.local` capability, which is a
**required capability**.  Every conformant PlayOS target provides it, so
an application may call `SavePath()` and `ConfigPath()` unconditionally
after `Lifecycle::Init()`.

See [Required Capabilities](../05-capability-model/03-required-capabilities.md).

## Public surface

```cpp
// include/playos/storage.h
namespace PlayOS {
namespace Storage {

// Absolute path for persistent save data.
// The directory is guaranteed to exist after the first call.
const char* SavePath();

// Absolute path for configuration data.
// The directory is guaranteed to exist after the first call.
const char* ConfigPath();

} // namespace Storage
} // namespace PlayOS
```

Both functions return a non-null, null-terminated UTF-8 string.  The returned
pointer remains valid for the lifetime of the process.

## Semantics

### `SavePath()`

Returns the root directory for **save data** — game progress, high scores,
unlocks, and any other state the player expects to persist across sessions.

The path is application-scoped on runtime devices (the runtime assigns a
unique base directory per package).  On SDK targets the backend uses a
conventional OS path (see *Platform implementations* below).

The directory MUST exist after the first call.  Subsequent calls return the
same pointer.

### `ConfigPath()`

Returns the root directory for **configuration data** — graphics settings,
key bindings, language preferences, and similar per-player options that are
distinct from game progress.

The separation from save data lets platforms back up or sync each type
independently.  The directory MUST exist after the first call.

## Usage example

```cpp
#include "playos/storage.h"
#include <cstdio>

void SaveScore(int score) {
    const char* dir = PlayOS::Storage::SavePath();
    char path[512];
    snprintf(path, sizeof(path), "%s/score.dat", dir);
    FILE* f = fopen(path, "wb");
    if (f) {
        fwrite(&score, sizeof(score), 1, f);
        fclose(f);
    }
}
```

Applications SHOULD use the paths as directory roots and create their own
sub-structure underneath (`saves/slot1/`, `saves/slot2/`, …).

## Platform implementations

### Linux (runtime devices and Linux SDK targets)

Follows the [XDG Base Directory Specification](https://specifications.freedesktop.org/basedir-spec/latest/):

| Function | Resolved path |
|---|---|
| `SavePath()` | `$XDG_DATA_HOME/PlayOS/saves` or `~/.local/share/PlayOS/saves` |
| `ConfigPath()` | `$XDG_DATA_HOME/PlayOS/config` or `~/.local/share/PlayOS/config` |

`$XDG_DATA_HOME` is honoured when set; the fallback is `$HOME/.local/share`.
A last-resort fallback of `./PlayOS` is used when `$HOME` is also unset
(e.g., in sandboxed environments).

Both directories are created with mode `0755` on first access.

> **Future (packaged apps):** When the runtime package model matures, each
> `.gpk` application will receive a *per-package* sub-directory under the
> platform base, e.g. `~/.local/share/PlayOS/saves/<package-id>/`.  The API
> surface stays identical; only the resolved path changes.

### Windows (Windows SDK target)

| Function | Resolved path |
|---|---|
| `SavePath()` | `%LOCALAPPDATA%\PlayOS\saves` |
| `ConfigPath()` | `%LOCALAPPDATA%\PlayOS\config` |

`SHGetFolderPath(CSIDL_LOCAL_APPDATA)` is used; `%USERPROFILE%\PlayOS` is the
fallback.  Directories are created with `CreateDirectoryA` on first access.

### Null (unit-test / headless SDK target)

Returns `"./PlayOS/saves"` and `"./PlayOS/config"` relative to the current
working directory.  No directories are created.  Use the null backend to test
path-construction logic without touching the filesystem.

## Design constraints

- **Read-only filesystem roots are not an error.** On a live-boot ISO the root
  filesystem is read-only (EROFS).  Save and config paths resolve into the
  tmpfs-backed overlay (`/var/lib/playos/…` after firstboot), which is always
  writable.
- **No engine dependency.** `storage.h` is a plain C++ header; it does not
  include Raylib, SDL, or any engine type.
- **Stable pointer lifetime.** The returned `const char*` is backed by a
  `static std::string` inside the backend instance.  Applications MUST NOT
  free it.
- **Thread safety.** The first call initialises an internal backend lazily.
  The first call SHOULD happen on the main thread during `Lifecycle::Init()`.
  Concurrent first calls are undefined.
