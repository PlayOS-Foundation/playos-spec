# Library

The **Library** is where players manage their installed applications.
It is accessible from the home screen.

## Layout

```
┌──────────────────────────────────────────┐
│  ← Library                         Sort ▾│
├──────────────────────────────────────────┤
│                                          │
│  [▼ Installed — 12 apps]                 │
│  [App 1] [App 2] [App 3] [App 4]        │
│  [App 5] [App 6] [App 7] [App 8]        │
│  [App 9] [App10] [App11] [App12]         │
│                                          │
│  [▼ Updates Available — 2 apps]          │
│  [App 3     Update 1.2 → 1.3]           │
│  [App 7     Update 2.0 → 2.1]           │
│                                          │
│  [▼ Download Queue — 1 app]              │
│  [App13    Downloading... 45% ████░░]   │
│                                          │
├──────────────────────────────────────────┤
│  [Y: Details]  [X: Uninstall]  [A: Launch]│
└──────────────────────────────────────────┘
```

## Sorting and filtering

Applications can be sorted by:

| Sort | Description |
|---|---|
| Recent | Most recently played first (default) |
| Title | Alphabetical by title |
| Size | By storage used |
| Installed | Most recently installed first |
| Updated | Most recently updated first |

## Application management

When an application is focused, contextual actions are available:

- **A** — Launch.
- **Y** — View details (version, size, permissions, release notes).
- **X** — Uninstall (with confirmation dialog).
- **Start** — Check for updates.

## Uninstall flow

1. Player presses X on an application.
2. Dialog: "Uninstall [App Name]? Your save data will be preserved."
   - Option: "Also delete save data" (checkbox, default off).
3. Player confirms.
4. Application is removed. Save data is kept in
   `/var/lib/playos/saves/<appId>/` unless the checkbox was selected.

## Updates

Updates are displayed as a separate section. Players can:

- **Update all** — start all pending updates.
- **Update individual** — update a specific application.
- **Pause/resume** — pause and resume downloads.
- **View changelog** — see what changed in the update.

## Storage management

The Library shows total storage usage at the top:

```
Storage: 234 GB free of 476 GB  [████████░░] 51% used
```

Applications are ordered by size in the "Sort by Size" view to help
players decide what to uninstall.

## Related

- [Home Screen](05-home-screen.md)
- [Downloads](10-downloads.md)
- [Updates (Package Format)](../10-package-format/13-updates-and-patches.md)
- [Uninstallation (Package Format)](../10-package-format/12-uninstallation.md)
