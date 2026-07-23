# Home Screen

The **Home Screen** is the first thing players see. It provides access
to applications, recently played titles, and system functions.

## Layout

```
┌──────────────────────────────────────────┐
│  [Time: 14:32]    [WiFi ▂▄▆█] [Batt 85%] │  Status bar
├──────────────────────────────────────────┤
│                                          │
│  ┌──────────────────────────────────────┐│
│  │  Featured / Recently Played          ││  Hero area
│  │  [Large game art]                    ││
│  │  "Continue Playing"                  ││
│  └──────────────────────────────────────┘│
│                                          │
│  [App 1]  [App 2]  [App 3]  [App 4]    │  Application grid
│  [App 5]  [App 6]  [App 7]  [App 8]    │
│  [App 9]  [App10]  [Store]  [Settings] │
│                                          │
├──────────────────────────────────────────┤
│  [A: Launch]  [Y: Details]  [≡: Menu]   │  Footer hints
└──────────────────────────────────────────┘
```

## Application grid

Applications are displayed in a scrollable grid. The grid:

- Shows application icon and title.
- Indicates recently updated (dot on icon).
- Indicates download progress (progress bar under icon).
- Sorts by most recently played by default.
- Supports manual reordering and folders.

## Navigation

- **D-pad / Left stick**: Move focus between applications.
- **A**: Launch focused application.
- **Y**: Open application details (info, updates, manage).
- **Start/Menu**: Open home menu (sort, create folder, system options).
- **Home button**: Always returns to the home screen.

## Quick resume

The most recently played application is highlighted in the hero area.
If the application supports suspend/resume, pressing A resumes it
instantly without a cold start.

## Search

Pressing X opens search. Type with the on-screen keyboard or a
connected keyboard. Search matches application titles and metadata.

## Related

- [Library](06-library.md)
- [Controller-First Navigation](03-controller-first-navigation.md)
- [Quick Resume (Suspend/Resume)](../08-runtime-architecture/16-suspend-and-resume.md)
