# Notification API

The Notification API lets applications post system notifications that appear
in the shell's notification center and (when configured) as toast overlays.
It is available on Runtime Devices and is a no-op on SDK Targets.

## Purpose

- **System notifications** — notify the player of events even when the
  application is not in focus (game invitations, friend online, download
  complete).
- **In-game toasts** — brief overlay messages (achievement unlocked,
  save complete).
- **Rich content** — notifications may include an icon, title, body, and
  optional action.

## API concepts

```cpp
namespace PlayOS {
namespace Notification {

struct Notification {
    const char* title;     // required
    const char* body;      // optional
    const char* iconPath;  // optional, path to a PNG
};

// Post a system notification. Returns immediately; the shell handles display.
void Post(const Notification& notification);

// Post a brief toast that auto-dismisses after ~3 seconds.
// Suitable for achievement popups, save confirmations.
void Toast(const char* message);

} // namespace Notification
} // namespace PlayOS
```

## Capability

Notifications are gated on the optional capability `platform.notifications`.
When absent (SDK Targets), `Post()` and `Toast()` are no-ops.

## Behavior

- `Post()` delivers a persistent notification to the shell's notification
  center. The player may dismiss it or interact with it.
- `Toast()` displays a brief overlay message (approximately 3 seconds) and
  does not persist in the notification center.
- Notifications from backgrounded applications are queued and displayed
  when the player returns to the shell.
- The application does not control display timing — the shell manages the
  notification queue.

## Requirements

- `Post()` MUST return immediately; notification display is asynchronous.
- Notifications MUST respect the player's notification preferences
  (do-not-disturb, per-app settings).
- `Toast()` messages MUST auto-dismiss without player interaction.
- The platform MUST rate-limit notifications to prevent spam (suggested:
  no more than 1 per second, 10 per minute).

## Related

- [Notifications](../09-shell-and-ux/11-notifications.md) — shell UX
- [Overlay API](15-overlay-api.md)
- [Achievements API](18-achievements-api.md) — achievements often trigger
  toasts
