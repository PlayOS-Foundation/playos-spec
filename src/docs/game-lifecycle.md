# Game Lifecycle

PlayOS is a console: the system owns the display and can background, suspend,
or terminate a game at any time. Games must poll lifecycle events every frame.

## Events

| Event | Meaning | Required behavior |
|---|---|---|
| `PLAYOS_LIFECYCLE_FOREGROUND` | Game is the active surface | Resume rendering/audio |
| `PLAYOS_LIFECYCLE_BACKGROUND` | Game is hidden | Pause simulation, mute audio, reduce CPU to near-zero |
| `PLAYOS_LIFECYCLE_SUSPEND` | System suspending | Save state immediately, return within 500 ms |
| `PLAYOS_LIFECYCLE_RESUME` | System resumed | Restore state |
| `PLAYOS_LIFECYCLE_TERMINATE` | Ordered shutdown | Save, release resources, `exit(0)` within 2 s |

## API

```c
PlayOSLifecycleEvent ev;
int r = playos_lifecycle_poll(&ev);       // non-blocking
int r = playos_lifecycle_wait(&ev, 100);  // blocking with timeout
int fd = playos_lifecycle_fd();            // for poll(2)/select(2)
```

## Rules

- Poll **every frame**; a missed TERMINATE can lose saves.
- On BACKGROUND, keep the process alive but idle — do not busy-loop.
- On SUSPEND, flush saves synchronously before returning.
- Never exit on BACKGROUND; only on TERMINATE.
