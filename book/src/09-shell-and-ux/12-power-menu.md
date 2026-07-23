# Power Menu

The **Power Menu** provides sleep, restart, and shutdown options. It
is accessed by long-pressing the power button or from the home screen.

## Layout

```
┌──────────────────────────────────────────┐
│                                          │
│  ┌──────────────────────────────────────┐│
│  │                                      ││
│  │         [Sleep Mode]                 ││
│  │         Instant resume               ││
│  │                                      ││
│  │         [Restart]                    ││
│  │         Full system restart          ││
│  │                                      ││
│  │         [Shut Down]                  ││
│  │         Power off completely         ││
│  │                                      ││
│  │         [Cancel]                     ││
│  │                                      ││
│  └──────────────────────────────────────┘│
│                                          │
└──────────────────────────────────────────┘
```

## Sleep mode

Sleep mode is the primary power action on handheld devices:

1. Player presses the power button (short press).
2. The Runtime suspends the current application.
3. Display, audio, and most peripherals are powered off.
4. RAM remains powered (or state is saved to disk for deep sleep).
5. On wake (power button press), the device resumes in ≤2 seconds.

Sleep mode draws minimal power — a device should last at least 48
hours in sleep on a full charge.

## Low battery behavior

| Battery Level | Action |
|---|---|
| ≤15% | Low battery notification |
| ≤10% | Battery saver mode (lower brightness, throttle CPU) |
| ≤5% | Critical alert. Auto-suspend if idle. |
| ≤3% | Emergency hibernate — save state and power off |

Emergency hibernate must complete within 5 seconds to prevent data
loss.

## Power button behavior

| Action | Response |
|---|---|
| Short press (while awake) | Sleep mode |
| Short press (while asleep) | Wake |
| Long press (2 seconds) | Power menu |
| Long press (10 seconds) | Hardware power off (failsafe) |

## Power menu from controller

The power menu can also be opened from the home screen or by pressing
and holding the Home button for 2 seconds. This ensures the power
menu is accessible without touching the physical power button.

## Related

- [Suspend and Resume](../08-runtime-architecture/16-suspend-and-resume.md)
- [Battery API](../06-platform-api/13-power-api.md)
- [Console-First Principle](../02-platform-principles/03-console-first.md)
