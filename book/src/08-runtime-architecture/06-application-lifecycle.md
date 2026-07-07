# Application Lifecycle

The runtime manages applications through a small, well-defined lifecycle.
Applications include games launched from the shell; the shell itself is a
long-lived application.

## States

```text
Installed
   ↓ launch
Launching
   ↓ ready
Running
   ↓ home / switch
Suspended            (optional — requires suspend capability)
   ↓ resume
Running
   ↓ exit
Exited → Return to Shell
```

- **Installed** — the package is present and executable.
- **Launching** — the runtime has started the process; the display is being
  handed to it.
- **Running** — the application owns the foreground surface and input.
- **Suspended** — the application is paused in the background (only where the
  `system.suspend` capability is present).
- **Exited** — the process has terminated; the runtime returns focus to the
  shell.

## Transitions and ownership

- Only one application holds the **foreground** (display + input focus) at a
  time; the compositor enforces this.
- On **Home**, the runtime either suspends the running game (if supported)
  or leaves it running in the background while the shell takes the
  foreground.
- On **exit**, the runtime reaps the process and restores the shell to the
  foreground.

## Requirements

- The runtime MUST return to the shell when a foreground application exits
  for any reason, including crashes.
- The runtime MUST NOT leave the display without an owner; if a game exits,
  the compositor reassigns the foreground to the shell.
- Suspend/resume behavior MUST be gated by the relevant capability.

## Vertical slice note

The proof-of-concept implements **Launching → Running → Exited → Return to
Shell**. Suspended state is deferred; on Home the slice may simply terminate
the game and return to the shell.

See: [Package Execution](07-package-execution.md),
[Game Switching](11-game-switching.md),
[Suspend and Resume](16-suspend-and-resume.md).
