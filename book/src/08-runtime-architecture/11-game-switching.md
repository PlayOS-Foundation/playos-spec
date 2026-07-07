# Game Switching

Game switching is the console loop: leave the shell, run a game, and come
back. It is the heart of the vertical slice.

## The loop

```text
Shell (foreground)
   ↓  user selects a game
Runtime launches game (see Package Execution)
   ↓  compositor gives foreground to the game
Game (foreground)
   ↓  user presses Home  /  game exits
Runtime returns foreground to the shell
   ↓
Shell (foreground)
```

The player experiences `Shell → Game → Shell`, exactly like a console
(`PS5 UI → Game → PS5 UI`).

## Switching models

Two behaviors are defined; availability depends on capabilities:

- **Stop-and-return (baseline).** On Home or exit, the game is terminated
  (or has already exited) and the shell returns to the foreground. Requires
  no special capability.
- **Suspend-and-resume (enhanced).** On Home, the game is paused in the
  background (`SIGSTOP`/cgroup freeze) and can be resumed. Requires the
  `system.suspend` capability. True instant resume of GPU/audio state is
  hardware- and application-dependent.

## Requirements

- The runtime MUST restore the shell to the foreground whenever the
  foreground game exits or is dismissed.
- Switching to another game MUST terminate or suspend the current one first;
  two games MUST NOT hold the foreground simultaneously.
- The compositor MUST reassign display ownership atomically so the display
  is never left without an owner.

## Vertical slice note

The proof-of-concept implements **stop-and-return**: select game → launch →
play → Home or exit → back to shell. Suspend-and-resume is deferred to a
later stage once the baseline loop is proven on the Ally.

See: [Application Lifecycle](06-application-lifecycle.md),
[Package Execution](07-package-execution.md),
[Input Routing](10-input-routing.md).
