# Shell

The **PlayOS Shell** is the system UI — the home screen, launcher,
settings, overlay, and all user-facing interface outside of
applications.

## Shell responsibilities

| Feature | Description |
|---|---|
| Home screen | Application grid, recent apps, quick resume |
| Launcher | Browse and launch installed applications |
| Overlay | Quick settings, notifications, performance HUD |
| Settings | System settings (network, audio, display, accounts) |
| Store | Browse, purchase, and install applications |
| Library | Manage installed applications |
| User profiles | Sign in, switch users, manage accounts |
| Notifications | System and application notifications |
| Controller pairing | Connect and manage input devices |

## Shell vs application UI

The Shell and applications have distinct rendering surfaces:

```
┌──────────────────────────────────────┐
│       Shell Surface (OpenGL)         │  ← Home screen, settings
├──────────────────────────────────────┤
│ Application Surface (OpenGL)         │  ← Game rendering
├──────────────────────────────────────┤
│  Overlay Surface (transparent)       │  ← Quick settings, notifications
├──────────────────────────────────────┤
│       System Compositor              │
├──────────────────────────────────────┤
│            Display                   │
└──────────────────────────────────────┘
```

One of Shell Surface or Application Surface is active at any time.
The Overlay Surface is always above both.

## Shell technologies

The reference Shell is built with **Raylib** for rendering and input.
This is a convenience choice, not a requirement. The Shell must:

- Be engine-agnostic (don't depend on Raylib structurally).
- Use the Platform API for input, display, and audio (same as
  applications).
- Not link against application code.

Alternative Shell implementations (Qt, SDL, Dear ImGui) are valid as
long as they conform to the Shell UX guidelines.

## Shell customization

OEMs can customize:

- **Visual theme** — colors, fonts, background.
- **Pre-installed applications** — curated content.
- **Store selection** — default OEM store.

OEMs cannot:

- Remove the PlayOS Marketplace store source.
- Block side-loading.
- Modify Platform API behavior.

## Related

- [Shell and UX](../09-shell-and-ux/01-overview.md)
- [Overlay Integration](../08-runtime-architecture/12-overlay-integration.md)
- [OEM Stores](../11-cloud-and-marketplace/18-oem-stores.md)
- [Reference Shell](../09-shell-and-ux/16-reference-shell.md)
