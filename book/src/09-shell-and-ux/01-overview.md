# Shell and UX Overview

The **PlayOS Shell** is the system UI that players interact with. It
provides the home screen, application launcher, settings, overlay, and
all system-level interaction outside of running applications.

## Design goals

1. **Controller-first** — every interaction must work with a gamepad.
   Touch and keyboard are secondary.
2. **TV-friendly** — readable from 3 meters with large text and clear
   contrast.
3. **Fast** — the Shell must start in under 3 seconds and respond to
   input within 16 ms.
4. **Accessible** — support screen readers, high contrast, remappable
   controls.
5. **Consistent** — the Shell establishes UX patterns that applications
   are encouraged to follow.

## Shell components

```
┌────────────────────────────────────┐
│        Shell Surface (OpenGL)      │
│                                    │
│  ┌──────────┐  ┌───────────────┐   │
│  │ Home     │  │ Settings      │   │
│  │ Screen   │  │ (overlay or   │   │
│  │          │  │  full-screen) │   │
│  └──────────┘  └───────────────┘   │
│                                    │
│  ┌──────────────────────────────┐  │
│  │  Overlay Surface (overlay)   │  │
│  │  Quick settings, notifs      │  │
│  └──────────────────────────────┘  │
│                                    │
└────────────────────────────────────┘
```

## Navigation model

Navigation is **spatial** (d-pad / left stick) with a focus highlight:

```
[App 1]  [App 2]  [App 3]
[App 4]  [App 5]  [App 6]
[App 7]  [App 8]  [App 9]
   ↑
  Focus
```

The d-pad moves focus. The A button activates. The B button goes back.
The home button returns to the home screen. This model is consistent
across the entire Shell.

## Chapter index

| Chapter | Description |
|---|---|
| [Console UX Principles](02-console-ux-principles.md) | Design rules for TV/controller UX |
| [Controller-First Navigation](03-controller-first-navigation.md) | Spatial focus model |
| [Touch Interaction](04-touch-interaction.md) | Touch adaptations |
| [Home Screen](05-home-screen.md) | Application grid and launcher |
| [Library](06-library.md) | Managing installed applications |
| [Store](07-store.md) | Browsing and purchasing applications |
| [Settings](08-settings.md) | System configuration |
| [Quick Settings](09-quick-settings.md) | Overlay quick settings panel |
| [Downloads](10-downloads.md) | Download management |
| [Notifications](11-notifications.md) | System notifications |
| [Power Menu](12-power-menu.md) | Sleep, restart, shutdown |
| [Developer Mode](13-developer-mode.md) | Dev tools and side-loading |
| [Accessibility](14-accessibility.md) | Screen reader, contrast, remapping |
| [Themes](15-themes.md) | Visual customization |
| [Reference Shell](16-reference-shell.md) | The Raylib-based reference Shell |
