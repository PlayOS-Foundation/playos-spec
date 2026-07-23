---
rfc: 0008
title: Game Lifecycle Contract
status: Accepted
authors: []
created: 2026-07-11
accepted: 2026-07-22
---

# RFC-0008: Game Lifecycle Contract

> **Status:** Accepted

## Summary

This RFC defines the precise contract between the PlayOS runtime and a
running application: how the runtime launches an application, what it
guarantees during execution, how it handles abnormal exits, and what the
application must do to behave correctly.  It formalises the state machine
sketched in `book/src/08-runtime-architecture/06-application-lifecycle.md`
with binding MUST/SHALL/SHOULD language.

## Motivation

`LaunchAndWait` works end-to-end in the vertical slice, but the contract
around crash handling, forced termination, exit codes, and the
shell-to-game-and-back transition is entirely implicit.  This creates
ambiguity for:

- compositor developers (when to reconfigure the surface),
- shell developers (how to interpret a non-zero exit code),
- game developers (what happens if they segfault),
- future package-sandbox authors (what signals are safe to send).

Without a written contract these decisions accumulate as tribal knowledge
and diverge across implementations.

## Guide-Level Explanation

From a game developer's perspective:

1. Your game binary is executed by the runtime.  `Lifecycle::Init()` gives
   you the Platform API.  `Lifecycle::Update()` keeps it alive every frame.
2. When the player presses **Home**, your game will eventually be asked to
   exit (Stage 1: immediately; Stage 2: after the overlay interaction).
3. When you return from `main()` with exit code `0`, the player sees the
   shell library again.
4. If your game crashes, the player also returns to the shell.  The runtime
   logs the event but does not punish the player.

## Reference-Level Explanation

### 1. Launch

The runtime SHALL launch the application executable in its own **process
group** (`setpgid(0, 0)` on POSIX, a new job object on Windows).

The runtime SHALL set the following environment variables before `exec`:

| Variable | Value |
|---|---|
| `WAYLAND_DISPLAY` | Name of the Wayland socket (e.g. `wayland-0`) |
| `XDG_RUNTIME_DIR` | Path to the runtime directory (e.g. `/run/playos`) |
| `PLAYOS_APP_ID` | The package identifier from the manifest (future) |

The runtime MUST NOT clear the inherited environment entirely; PATH and
locale variables MUST be preserved.

### 2. Running state

While the application is running:

- The runtime MUST NOT send `SIGKILL` or `SIGTERM` except as part of forced
  exit (§4).
- The runtime MUST route Wayland events to the application's surface while
  it holds the foreground.
- The runtime MUST NOT start a second foreground application while the first
  is running; requests to launch are queued or rejected.

### 3. Normal exit

A **normal exit** occurs when the application's main process exits with any
code.

- The runtime MUST wait for the process to exit (`waitpid`).
- Exit code `0` SHOULD be interpreted as clean exit.
- Exit codes `1`–`125` MAY be surfaced to the shell as diagnostic
  information.
- Exit code `127` is reserved for "executable not found" (exec failure).
- The runtime MUST restore the shell to the foreground within **500 ms** of
  detecting exit.

### 4. Forced exit (Home button, timeout)

If the system requests the application to exit (Home-button action, or a
future watchdog timeout):

1. The runtime SHOULD send `SIGTERM` to the process group.
2. The runtime SHALL wait up to **3 seconds** for the process to exit.
3. If the process has not exited after 3 seconds, the runtime SHALL send
   `SIGKILL` to the process group.
4. The runtime MUST restore the shell to the foreground after the process is
   gone.

> **Stage 1:** Step 1 is the only step implemented.  The 3-second grace
> period and SIGKILL are deferred.

### 5. Crash

A **crash** is any unexpected exit: signal termination (SIGSEGV, SIGABRT,
…), abort, or OOM kill.

- The runtime MUST detect the crash via `waitpid` status.
- The runtime MUST restore the shell to the foreground.
- The runtime SHOULD log the exit status and signal number to the platform
  log (`/var/log/playos/`).
- The runtime MUST NOT display a blocking error dialog on behalf of the
  application; the shell MAY show a non-blocking notification.

### 6. Application requirements

An application that conforms to the PlayOS lifecycle contract:

- MUST call `PlayOS::Lifecycle::Init()` before any other Platform API call.
- MUST call `PlayOS::Lifecycle::Update()` once per frame for the duration of
  its run.
- MUST call `PlayOS::Lifecycle::Shutdown()` before returning from `main()`.
- MUST handle `SIGTERM` gracefully (flush save data, exit within 3 seconds).
- MUST NOT call `_exit()` or `abort()` in normal operation.
- SHOULD exit with code `0` on clean termination.

### 7. Exit code registry

| Code | Meaning |
|---|---|
| `0` | Clean exit |
| `1` | Generic error (application-defined) |
| `2`–`125` | Application-defined |
| `126` | Permission denied (exec failed) |
| `127` | Executable not found |
| `128 + N` | Killed by signal N (POSIX shell convention) |

## Drawbacks

- Defining a 3-second grace period creates a worst-case 3-second black
  screen if a game hangs.  A shorter timeout risks killing games that are
  doing legitimate slow saves.
- A watchdog introduces runtime complexity and a small per-frame overhead
  for heartbeat tracking.

## Rationale and Alternatives

- **No forced-exit timeout**: leaves the system in a permanently black
  state if a game deadlocks.  Rejected.
- **Cooperative suspend instead of SIGTERM**: requires a Wayland protocol
  extension.  Deferred to Stage 2 (suspend-and-resume, see Future
  Possibilities).
- **Immediate SIGKILL on Home**: fast but risks data loss.  The 3-second
  grace period balances safety with responsiveness.
- **Game-managed exit only (no runtime authority to kill)**: violates the
  console-first principle — the platform, not the game, owns the
  foreground.
- **Per-device timeout values**: considered for slow-storage devices (SD
  cards).  Rejected in favour of a single timeout with the ability for
  device profiles to extend it via `game.graceful_exit_timeout_ms`.
  The default of 3000 ms covers the common case; device profiles may
  increase it.

## Prior Art

- **Android Activity lifecycle** (`onPause`, `onStop`, `onDestroy`):
  provides a structured state machine with guaranteed callbacks before
  termination.  PlayOS adopts a similar callback model (`OnSuspend` in
  Stage 2) but keeps the Stage 1 contract deliberately simpler.
- **Steam Deck / SteamOS**: uses `SIGTERM` → `SIGKILL` for game
  termination via the Steam client, with a configurable timeout.
- **Nintendo Switch**: the OS suspends the game process on Home; games
  do not receive a termination signal unless the user explicitly closes
  the title.  PlayOS Stage 2 aims for this model.
- **POSIX process groups**: the `setpgid` + signal-to-group pattern is
  well-established (systemd, supervisord).  PlayOS follows this
  convention.

## Unresolved Questions

None — all questions from the draft phase have been resolved.

- **Exit code surfacing:** Exit codes MAY be surfaced in the Shell UI
  as non-blocking diagnostic information in developer mode.  In
  production, non-zero exit codes SHOULD be logged but MUST NOT display
  a blocking error dialog.  The Shell MAY show a non-intrusive
  notification ("Game closed unexpectedly").
- **Slow-storage timeout:** The 3-second default covers the common case.
  Device profiles MAY override via the `game.graceful_exit_timeout_ms`
  field.  The minimum allowed value is 1000 ms; there is no maximum.
- **Watchdog for stalled games:** The runtime SHOULD implement a
  heartbeat watchdog.  The Lifecycle Manager monitors the
  `Lifecycle::Update()` call frequency.  If `Update()` is not called for
  5 consecutive seconds, the runtime treats the application as hung and
  initiates forced exit (§4).  The watchdog is a SHOULD, not a MUST —
  it may be disabled in developer mode.

## Future Possibilities

- **Suspend-and-resume (Stage 2):** Instead of SIGTERM on Home, the
  runtime suspends the game process (`SIGSTOP` / cgroup freeze) and
  resumes it when the player returns.  Requires the `system.suspend`
  capability and a Wayland protocol extension for surface hand-off.
  See [Suspend and Resume](../book/src/08-runtime-architecture/16-suspend-and-resume.md).
- **Multi-application foreground:** Future compositor support for
  picture-in-picture or split-screen could allow two applications to
  share the foreground.  The lifecycle contract would need to define
  "partial foreground" states.
- **Background applications:** Applications that continue running
  (e.g., music players, download managers) while not holding the
  foreground.  Would require a new lifecycle state and capability.
- **Crash recovery:** Automatic restart of a crashed game (with player
  consent) if save data was recently written.  Depends on the
  observability and telemetry RFC.

## Implementation notes

- `playos-runtime/src/process_posix.cpp`: `LaunchAndWait` currently does not
  send SIGTERM; forced-exit is done by the shell's `timeout` mechanism in the
  vertical slice.
- `playos-runtime/compositor/playos-compositor.c`: currently no IPC to
  signal a running game to exit; the Home-button action is TODO.
