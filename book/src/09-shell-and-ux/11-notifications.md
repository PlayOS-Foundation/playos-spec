# Notifications

The **Notifications** system delivers alerts from the system, cloud
services, and applications to the player.

## Notification sources

| Source | Examples |
|---|---|
| **System** | Update available, low battery, storage low |
| **Cloud** | Friend online, achievement unlocked, sale on wishlisted app |
| **Applications** | Game invite, daily reward, event starting |

## Delivery

Notifications appear in three places:

1. **Overlay toast** — brief popup in the top-right corner during
   gameplay (auto-dismiss after 5 seconds).
2. **Quick Settings** — notifications list in the Quick Settings
   panel.
3. **Full notification center** — accessible from the home screen.

## Toast format

```
┌─────────────────────────────────────┐
│ [App Icon]  Game Title              │
│             Daily challenge ready!  │
│                          [A: Open]  │
└─────────────────────────────────────┘
```

Toasts auto-dismiss after 5 seconds. Pressing A on a toast opens the
relevant application or view.

## Notification center

```
┌──────────────────────────────────────────┐
│  ← Notifications                   Clear │
├──────────────────────────────────────────┤
│                                          │
│  Today                                   │
│  ┌──────────────────────────────────────┐│
│  │ [🔧] System Update Available         ││
│  │      Version 0.5.0 is ready.         ││
│  │                     2 hours ago      ││
│  ├──────────────────────────────────────┤│
│  │ [👤] Friend Online                    ││
│  │      PlayerName is now online.       ││
│  │                     3 hours ago      ││
│  └──────────────────────────────────────┘│
│                                          │
│  Yesterday                               │
│  ┌──────────────────────────────────────┐│
│  │ [🏆] Achievement Unlocked             ││
│  │      "First Victory" in Game Title   ││
│  │                     21 hours ago     ││
│  └──────────────────────────────────────┘│
│                                          │
└──────────────────────────────────────────┘
```

## Do Not Disturb

Players can enable "Do Not Disturb" in Quick Settings. When enabled:

- Toasts are suppressed.
- Notifications are still delivered to the notification center.
- Critical alerts (low battery ≤5%) still appear.

## Notification permissions

Applications that want to send notifications must declare
`notifications.send` in their manifest. Players can disable
notifications per-application in Settings.

## Related

- [Notification API](../06-platform-api/21-notification-api.md)
- [Quick Settings](09-quick-settings.md)
- [Overlay Integration](../08-runtime-architecture/12-overlay-integration.md)
- [Permissions](../10-package-format/09-permissions.md)
