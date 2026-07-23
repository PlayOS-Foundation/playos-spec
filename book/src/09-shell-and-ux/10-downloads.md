# Downloads

The **Downloads** screen manages active and queued downloads. It is
accessible from the home screen and from the Store.

## Layout

```
┌──────────────────────────────────────────┐
│  ← Downloads                             │
├──────────────────────────────────────────┤
│                                          │
│  Active Downloads                        │
│  ┌──────────────────────────────────────┐│
│  │ [App Icon] Game Title                ││
│  │ Version 1.2.0   2.3 GB               ││
│  │ ████████░░░░░░  52%  1.2 GB / 2.3 GB ││
│  │ 8 MB/s  ·  4 min remaining           ││
│  │ [⏸ Pause]  [✕ Cancel]               ││
│  └──────────────────────────────────────┘│
│                                          │
│  Queued                                  │
│  ┌──────────────────────────────────────┐│
│  │ [App Icon] Other Game                ││
│  │ Version 1.0.0   1.1 GB               ││
│  │ Waiting...                           ││
│  │ [▶ Start Now]  [✕ Remove]           ││
│  └──────────────────────────────────────┘│
│                                          │
│  Completed                               │
│  ┌──────────────────────────────────────┐│
│  │ [App Icon] Installed App             ││
│  │ Version 3.4.1   Installed ✓          ││
│  │ [A: Launch]  [✕ Clear]              ││
│  └──────────────────────────────────────┘│
│                                          │
└──────────────────────────────────────────┘
```

## Download management

- **Concurrent downloads**: Up to 3 simultaneous downloads.
- **Priority**: Foreground (player-initiated) downloads take priority
  over background (automatic update) downloads.
- **Pause/Resume**: Downloads can be paused and resumed. Progress is
  saved across reboots.
- **Background**: Downloads continue when the player is in another
  application (unless the application uses the network heavily).

## Download policies

| Policy | Behavior |
|---|---|
| Wi-Fi only | Only download on Wi-Fi (default for large apps) |
| Allow cellular | Download on any connection |
| Auto-update | Download and install updates automatically |
| Notify only | Notify when updates are available, don't download |
| Metered connection | Warn before downloading on metered connections |

## Error handling

If a download fails:

- Retry up to 3 times with exponential backoff.
- If all retries fail, show error with option to retry or cancel.
- Partial downloads are preserved for resume.
- Corrupted downloads are detected via content hash verification.

## Progress indication

Download progress is visible:

- **Download screen**: Detailed progress with speed and ETA.
- **Home screen**: Progress bar under application icon.
- **Notification**: Compact progress notification when downloads are
  backgrounded.

## Related

- [Store](07-store.md)
- [Library](06-library.md)
- [Updates (Runtime)](../08-runtime-architecture/15-updates.md)
- [Updates and CDN](../11-cloud-and-marketplace/16-updates-and-cdn.md)
