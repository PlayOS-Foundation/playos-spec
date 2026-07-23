# Overlay Integration

The **overlay** is system UI drawn on top of a running application. It
provides quick settings, notifications, and system status without
interrupting gameplay.

## What the overlay provides

| Feature | Description |
|---|---|
| Quick Settings | Brightness, volume, Wi-Fi, Bluetooth, flight mode |
| Notifications | Toast messages from the system or other apps |
| Game Switching | Switch between running applications |
| Home Button | Return to the Shell home screen |
| Status Bar | Battery, time, network, notifications badge |
| Performance Overlay | FPS counter, CPU/GPU usage (dev mode) |

## How it works

```
┌──────────────────────────────────┐
│        Application (OpenGL)      │  ← Renders to its own framebuffer
├──────────────────────────────────┤
│   Overlay Surface (transparent)  │  ← System compositor layer
├──────────────────────────────────┤
│         System Compositor        │  ← Composites app + overlay
├──────────────────────────────────┤
│              Display             │
└──────────────────────────────────┘
```

The application renders to its own framebuffer. The system compositor
composites the application framebuffer with the overlay surface. The
application does not need to know the overlay exists.

## Triggering the overlay

The overlay is opened and closed by a **system button** (e.g., the
ROG Ally Command Center button). The button is configured in the
device profile:

```toml
[input]
quickSettingsButton = "asus_command_center"
```

The Runtime captures this button before it reaches the application.
The application does not receive the button press.

## Application awareness

Applications can query whether the overlay is open:

```cpp
if (Overlay::IsVisible()) {
    // Pause gameplay, mute audio, etc.
}
```

Applications receive an `OverlayStateChanged` event so they can react:

```cpp
// In the game loop:
if (Overlay::StateChanged()) {
    if (Overlay::IsVisible()) {
        PauseGame();
    } else {
        ResumeGame();
    }
}
```

## Capability

Overlay support is gated by `system.overlay`. On SDK Targets, this
capability is absent.

## Drawing restrictions

- The overlay **must not** be used for in-game UI. Applications should
  draw their own UI.
- The overlay is always on top. Applications cannot draw over it.
- The overlay uses a fixed resolution independent of the application.

## Performance

- The overlay is rendered by the system compositor, not the
  application.
- Opening the overlay does not consume the application's GPU budget.
- The compositor caches the application's last frame while the overlay
  is open.

## Related

- [Overlay API](../06-platform-api/15-overlay-api.md)
- [Runtime Services](08-runtime-services.md)
- [Game Switching](11-game-switching.md)
- [Quick Settings Panel](../09-shell-and-ux/09-quick-settings.md)
