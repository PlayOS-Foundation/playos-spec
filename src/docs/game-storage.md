# Game Storage

`libplayos` provides per-game, isolated storage paths. Games must not
hard-code `/data` paths.

## Paths

| API | Purpose | Writable |
|---|---|---|
| `playos_storage_get_install_path()` | Game installation files | read-only |
| `playos_storage_get_saves_path()` | Save data / progress | yes |
| `playos_storage_get_cache_path()` | Cache (cleared by user anytime) | yes |
| `playos_storage_get_games_path()` | All games root (shell/library use) | read-only |

All returned strings are valid for the process lifetime — never `free()` them.

## Crash-safe saves

```c
char path[512];
snprintf(path, sizeof(path), "%s/save.bin", playos_storage_get_saves_path());

const char *data = "player progress";
playos_storage_atomic_write(path, data, strlen(data));   // write + rename
playos_storage_atomic_replace(tmp_path, path);           // move into place
```

`atomic_write` is crash-safe: either the old file or the complete new file is
present after a crash.

## Rules

- Saves go in `saves_path`; cache goes in `cache_path` — never the reverse.
- The user can clear cache; do not store progress there.
- Check `playos_storage_free_bytes()` before large writes.
