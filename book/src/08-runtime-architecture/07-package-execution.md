# Package Execution

Package execution is how the runtime launches and supervises an application
process. The vertical slice depends on this: the shell must launch a game
and know when it exits.

## Launch model

The runtime launches an application as a **child process in its own process
group**, so the whole application can be signalled and reaped as a unit.

```text
Shell requests launch (application id)
   ↓
Runtime resolves the executable and working directory
   ↓
Runtime starts the process in a new process group / scope
   ↓
Compositor gives the foreground to the new client
   ↓
Runtime monitors the process
   ↓
On exit: runtime reaps it and returns foreground to the shell
```

On the reference runtime, processes are started as **systemd user scopes**
(`systemd-run --user --scope`) or an equivalent, which provides clean
cgroup-based tracking and teardown.

## Supervision

- The runtime tracks the process (and its group) for lifetime and exit
  status.
- On exit or crash, the runtime MUST reap the process and notify the shell
  so it can restore the foreground.
- Terminating an application MUST terminate its whole process group, not
  just the leader.

## Isolation (future)

Stronger isolation (sandboxing, permission enforcement) is specified in the
security part and is not required for the vertical slice.

## Vertical slice note

Minimal implementation: the shell asks the runtime to launch a demo game;
the runtime `fork/exec`s (or `systemd-run --scope`s) the executable in its
own process group, waits for exit, and signals the shell to return to the
foreground. Signing, permissions, and sandboxing are out of scope for the
first slice.

See: [Application Lifecycle](06-application-lifecycle.md),
[Game Switching](11-game-switching.md),
[Package Format](../10-package-format/01-overview.md).
