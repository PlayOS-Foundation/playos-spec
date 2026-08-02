# Sprint 7 — Game Launch, Lifecycle, System Button, and Overlay

**Goal:** Implement the complete console lifecycle: launch a game, background it with the System button, show the PlayOS overlay, resume the game, handle clean exit, and recover from crashes. The public `playos-platform-api` delivers lifecycle events to games; `playos-runtime` keeps these paths trusted and private.

**Primary Outcome:** A real game launches from the shell, runs with hardware acceleration, the System button surfaces the overlay, resume returns to the same running game, and both clean exit and crash return to the shell without any black screen or visible Linux prompt.

**Prerequisites:** Sprint 6 complete — persistent storage and game discovery working.

---

## Why This Sprint Exists

Sprint 6 established a working game library. Sprint 7 turns selection into a real console experience: a game must launch, run exclusively, suspend to the system overlay, resume, and handle crashes — without ever revealing a Linux terminal. This sprint also pins the complete compositor state machine and private Wayland protocol, which subsequent sprints depend on.

---

## Start Condition Checklist

- Sprint 6 complete: game library populated, manifest discovery working.
- `playos-compositor` renders the shell surface and can switch surfaces in principle.
- `playos-platform-api` lifecycle stubs exist but are not backed by real delivery.
- `playos-overlay` binary does not yet exist.
- `PLAYOS_BUTTON_SYSTEM` hardware mapping confirmed (from Sprint 3 `evtest` work).

---

## Decisions Locked for This Sprint

- **One-game rule:** only one game process may run at a time; a second launch request is rejected with a clear error log
- **First-frame rule:** compositor does NOT switch display to the game until the game commits its first Wayland buffer
- **System button intercept:** `PLAYOS_BUTTON_SYSTEM` is consumed by the compositor seat — it is NEVER forwarded to any client
- **Lifecycle delivery:** via file descriptor (`PLAYOS_LIFECYCLE_FD`) passed at launch; games must not poll any other channel
- **Non-cooperative fallback:** `SIGSTOP`/`SIGCONT` — only `playos-init` sends signals; compositor requests via IPC
- **Crash recovery deadline:** display must return to shell within 500ms of game process exit
- **Overlay z-order:** game surface → dim layer → overlay surface — this order is locked

---

## Scope

### In Scope

- Full game launch flow (shell → init → compositor → game)
- Complete compositor state machine
- Reserved system button intercept
- `playos-overlay` trusted client (minimal: resume, quit, battery, thermal)
- Real lifecycle event delivery via `PLAYOS_LIFECYCLE_FD`
- Game exit (clean and crash) with shell recovery
- Private Wayland protocol Sprint 7 additions
- Non-cooperative game SIGSTOP fallback

### Explicitly Out of Scope

- Audio (Sprint 8)
- Power/thermal display (Sprint 9 detail — battery % stub is acceptable this sprint)
- Installer (Sprint 10)
- A/B updates (Sprint 11)

---

## Required Repository Changes

| Repo | Required work |
|---|---|
| `playos-compositor` | Full state machine, first-frame rule, system button intercept, overlay surface management |
| `playos-shell` | Launch IPC, launching-state spinner UI, crash notification, library restore |
| `playos-runtime` | Launch IPC transport, `FactoryReset` additions, private Wayland protocol extensions, `playos-overlay` package |
| `playos-platform-api` | Real lifecycle event delivery (fd-backed) |
| `playos-refdistro` | `playos-overlay` Buildroot package |

---

## Expected Files and Directories

### `playos-compositor`

```text
src/state_machine.c       # all states and transitions
src/system_button.c       # input intercept at seat level
src/overlay_manager.c     # z-order, show/hide logic
```

### `playos-runtime`

```text
protocols/playos-v1.xml   # Sprint 7 additions (compositor_control, overlay_v1)
src/lifecycle_transport.c # fd creation, event write
src/overlay_process.c     # spawn/manage overlay client
```

### `playos-refdistro`

```text
br2-external/package/playos-overlay/
    playos-overlay.mk
    Config.in
src/playos-overlay/
    main.c                # Raylib overlay UI
```

---

## Agent Task Breakdown

### Task Status Grid

| Task ID | Task | Primary repo | Status | Notes / evidence |
|---|---|---|---|---|
| S7-T1 | Implement full game launch flow in `playos-init` | `playos-refdistro` | not started | |
| S7-T2 | Implement full compositor state machine | `playos-compositor` | not started | |
| S7-T3 | Implement system button intercept at seat level | `playos-compositor` | not started | |
| S7-T4 | Build `playos-overlay` trusted Raylib client | `playos-runtime`, `playos-refdistro` | not started | |
| S7-T5 | Implement lifecycle fd delivery in platform-api | `playos-platform-api` | not started | |
| S7-T6 | Extend private Wayland protocol (Sprint 7 interfaces) | `playos-runtime` | not started | |
| S7-T7 | Implement game exit and crash recovery | `playos-compositor`, `playos-refdistro` | not started | |
| S7-T8 | Integration and lifecycle validation on Ally | `playos-refdistro` | not started | |

### S7-T1 — Implement full game launch flow in `playos-init`

1. Validate: one-game rule; reject immediately if a game is already running
2. Validate: manifest exists, executable exists, `api_version` ≤ current system version
3. Prepare environment variables: `PLAYOS_GAME_ID`, `PLAYOS_INSTALL_PATH`, `PLAYOS_SAVE_PATH`, `PLAYOS_CACHE_PATH`, `WAYLAND_DISPLAY=playos-0`, `PLAYOS_LIFECYCLE_FD`, `PLAYOS_LAUNCH_TOKEN`
4. Create lifecycle pipe; store write-end in `playos-init`, pass read-end as `PLAYOS_LIFECYCLE_FD`
5. Spawn game executable; track PID
6. Emit `GameStarted { game_id, pid, launch_token }` to compositor and shell via runtime IPC

**Done when:** `com.playos.sample-input` launches from the shell, receives the environment variables, and a `GameStarted` event is logged in the compositor.

### S7-T2 — Implement full compositor state machine

Implement all five states and all transitions:

```
SHELL_FOREGROUND
  → (LaunchGame accepted)         → GAME_STARTING
GAME_STARTING
  → (first committed buffer)      → GAME_FOREGROUND
  → (launch timeout/crash)        → SHELL_FOREGROUND
GAME_FOREGROUND
  → (PLAYOS_BUTTON_SYSTEM)        → PLAYOS_UI_FOREGROUND_WITH_GAME_BACKGROUND
  → (game process exits)          → SHELL_FOREGROUND
PLAYOS_UI_FOREGROUND_WITH_GAME_BACKGROUND
  → (overlay Resume)              → GAME_FOREGROUND
  → (overlay Quit → TerminateGame → game exits) → SHELL_FOREGROUND
SHELL_FOREGROUND (after game)
  → shell surface unfocused, restored
```

Log all transitions: `state_machine: GAME_STARTING → GAME_FOREGROUND`.

**Done when:** state logs are visible matching all transition paths in the test matrix.

### S7-T3 — Implement system button intercept at seat level

- Register an input filter at the wlroots seat level that consumes `PLAYOS_BUTTON_SYSTEM` before any client can receive it
- When in `GAME_FOREGROUND`: remove input focus from game, write `PLAYOS_LIFECYCLE_BACKGROUND` to lifecycle fd, transition to overlay state
- Verify via `evtest` and a game that logs all key events — `PLAYOS_BUTTON_SYSTEM` must never appear in the game log

**Done when:** pressing the system button in-game activates the overlay and the game never sees the key event.

### S7-T4 — Build `playos-overlay` trusted Raylib client

Initial overlay content:
- Game title and elapsed running time
- "Resume Game" (A button) — sends resume request to compositor via private protocol
- "Quit Game" (B or menu) — sends `TerminateGame` IPC to `playos-init`
- Battery percentage (stub value acceptable this sprint)
- Thermal status (stub value acceptable this sprint)

Surface policy:
- Pre-spawned at boot, hidden; compositor shows/hides it via `playos_overlay_v1`
- Overlay uses `playos_overlay_v1::register_as_overlay` to identify itself
- Overlay must implement `about_to_show` / `about_to_hide` to reset state cleanly

**Done when:** overlay is visible above a running game when system button is pressed, and both Resume and Quit work.

### S7-T5 — Implement lifecycle fd delivery in platform-api

Replace stub `playos_lifecycle_poll` with a real implementation:

```c
/* Non-blocking. Returns 1 if event available, 0 if none, -1 on error. */
int playos_lifecycle_poll(PlayOSLifecycleEvent *event);
```

- Read from `PLAYOS_LIFECYCLE_FD` in non-blocking mode
- Deserialize the event type (single-byte or small struct)
- Support: `PLAYOS_LIFECYCLE_FOREGROUND`, `PLAYOS_LIFECYCLE_BACKGROUND`, `PLAYOS_LIFECYCLE_TERMINATE`

Non-cooperative fallback: if `playos-init` receives no CPU reduction signal within 500ms of sending `BACKGROUND`, it sends `SIGSTOP` to the game PID. On resume, it sends `SIGCONT`.

**Done when:** `com.playos.sample-input` logs lifecycle events it receives and the sequence is correct across the full launch/background/resume/quit cycle.

### S7-T6 — Extend private Wayland protocol

Add to `playos-runtime/protocols/playos-v1.xml`:

```xml
<interface name="playos_compositor_control_v1" version="1">
  <request name="set_expected_game">
    <arg name="launch_token" type="string"/>
  </request>
  <request name="terminate_game"/>
  <request name="show_overlay"/>
  <request name="hide_overlay"/>
</interface>

<interface name="playos_overlay_v1" version="1">
  <request name="register_as_overlay"/>
  <event name="about_to_show"/>
  <event name="about_to_hide"/>
</interface>
```

Regenerate Wayland scanner outputs. Only `playos-runtime` processes may use `playos_compositor_control_v1`.

**Done when:** compositor binds both interfaces; overlay registers itself; CI protocol scanner passes.

### S7-T7 — Implement game exit and crash recovery

**Clean exit:**
1. `playos-init` records exit status; closes lifecycle pipe write-end
2. Compositor detects Wayland client disconnect; transitions to `SHELL_FOREGROUND`
3. Shell surface is unhidden and refocused; library scroll position restored
4. Shell shows no notification on clean exit

**Crash (non-zero exit or signal):**
1. Same compositor recovery
2. Shell shows a non-intrusive toast notification: "Game exited unexpectedly"
3. Options: "Restart" (re-sends `LaunchGame`) or "Back to Library" (dismiss)

**Invariant:** A game crash must NEVER reveal a Linux terminal, leave the display black for >500ms, or require reboot.

**Done when:** `kill -9 <game_pid>` while the game is running returns display to the shell within 500ms with the crash toast visible.

### S7-T8 — Integration and lifecycle validation

Run the full test matrix on the ROG Ally:

1. Launch → play → quit via Quit button → library shown
2. Launch → system button → overlay visible → resume → back in game
3. Launch → system button → overlay visible → quit → library shown
4. `kill -9 <game_pid>` → crash toast within 500ms
5. Try to launch a second game while one is running → reject logged, first game unaffected
6. Launch → background → game ignores lifecycle → `SIGSTOP` fires within 500ms
7. Repeat items 1–3 three cycles in a row → no state leaks

**Done when:** all 7 test cases pass on the Ally with evidence logged.

---

## Implementation Guidance

### Launch token

The launch token is a one-time random UUID generated per launch. It expires when the first Wayland client presents with that token. Clients presenting an unknown or expired token during the launch window are rejected and logged, not crashed.

### Display switch timing

The shell surface must remain visible until the game's first Wayland buffer is committed. Do not hide the shell on `GameStarted` — hide it only on the first-frame transition.

### Overlay pre-spawn

`playos-overlay` must be spawned at compositor startup, not on first use. This avoids visible latency when the system button is pressed. Map/unmap the surface; do not spawn/kill the process per use.

---

## Verification and Evidence

| Evidence | How it is produced |
|---|---|
| State machine logs | compositor systemd journal showing all transitions for the test matrix |
| Lifecycle event log | `com.playos.sample-input` log showing FOREGROUND/BACKGROUND/FOREGROUND sequence |
| System button intercept proof | game key log showing system button event is absent |
| Crash recovery timing | timestamp of game exit vs. timestamp of shell surface shown (≤500ms) |
| One-game rule | log showing second launch rejected |
| `SIGSTOP` fallback | `strace` or log showing signal sent within 500ms |

---

## Acceptance Criteria

- [ ] Selecting a game launches it; first-frame switch happens with no black flash
- [ ] Game receives `PLAYOS_LIFECYCLE_FOREGROUND` at launch
- [ ] System button press shows overlay above game (dim + overlay visible)
- [ ] Game receives `PLAYOS_LIFECYCLE_BACKGROUND` on system button press
- [ ] Overlay "Resume" returns to game; `PLAYOS_LIFECYCLE_FOREGROUND` delivered
- [ ] Overlay "Quit" terminates game cleanly; shell shown
- [ ] Clean exit returns to shell with library scroll position restored
- [ ] `kill -9 <game_pid>` returns display to shell within 500ms; crash toast shown
- [ ] Second launch attempt while game is running is rejected; first game unaffected
- [ ] `PLAYOS_BUTTON_SYSTEM` never appears in any game client's input stream
- [ ] Non-cooperative game receives `SIGSTOP` within 500ms of `BACKGROUND` event
- [ ] Full 3-cycle lifecycle test passes without state leaks

---

## Handoff to Sprint 8

Sprint 8 may assume:

- Full lifecycle (launch, background, resume, terminate) works on the Ally
- `PLAYOS_LIFECYCLE_FD` delivers events reliably
- The overlay exists and can be extended (add audio volume controls)
- `rcore_playos.c` is already in `playos-shell` from Sprint 5; Sprint 8 adds the ALSA backend to it
- The compositor state machine is stable and not to be modified in Sprint 8

---

## Exit Gate

The complete console lifecycle works on the ROG Ally: launch, system button overlay, resume, clean exit, and crash recovery all behave correctly. The player never sees a Linux prompt.

*Previous: [Sprint 6](Sprint-6.md) | Next: [Sprint 8](Sprint-8.md)*

**`playos-shell` side:**
1. Player presses A on a game → shell enters Launching state (spinner UI)
2. Shell sends `LaunchGame { game_id, manifest_path }` over control IPC
3. Shell hides or throttles its rendering while game starts

**`playos-init` side:**
1. Validate: one-game rule (reject if game already running)
2. Validate: manifest exists, executable exists, `api_version` supported
3. Prepare per-game environment:
   ```
   PLAYOS_GAME_ID=<game-id>
   PLAYOS_INSTALL_PATH=/data/games/<game-id>
   PLAYOS_SAVE_PATH=/data/saves/<game-id>
   PLAYOS_CACHE_PATH=/data/cache/<game-id>
   WAYLAND_DISPLAY=playos-0
   PLAYOS_LIFECYCLE_FD=<fd>     # lifecycle event pipe
   PLAYOS_LAUNCH_TOKEN=<uuid>   # one-time identity token
   ```
4. Spawn game executable; track PID
5. Emit `GameStarted { game_id, pid, launch_token }` event to compositor and shell

**`playos-compositor` side:**
1. Receive `GameStarted` event from runtime transport
2. Store `expected_launch_token`
3. When a new Wayland client connects: check its `PLAYOS_LAUNCH_TOKEN` environment
4. Reject clients with no or wrong token during the launch window
5. Wait for the game's **first committed buffer** (first-frame rule)
6. Only then: transition state to `GAME_FOREGROUND`; hide the shell surface

### Compositor State Machine (Full Implementation)

Implement all states from the architecture:

```
SHELL_FOREGROUND
  → (launch accepted)         → GAME_STARTING
GAME_STARTING
  → (first valid frame)       → GAME_FOREGROUND
GAME_FOREGROUND
  → (PLAYOS_BUTTON_SYSTEM)    → PLAYOS_UI_FOREGROUND_WITH_GAME_BACKGROUND
  → (game exit or crash)      → SHELL_FOREGROUND
PLAYOS_UI_FOREGROUND_WITH_GAME_BACKGROUND
  → (Resume)                  → GAME_FOREGROUND
  → (Quit → TerminateGame)    → TERMINATING_GAME → SHELL_FOREGROUND
```

### Reserved System Button

`PLAYOS_BUTTON_SYSTEM` is mapped to the ROG Ally's Armory Crate / ROG button (identify via `evtest` in Sprint 3).

**Compositor intercepts this key at the seat level — it is never forwarded to any client.**

When pressed in `GAME_FOREGROUND`:
1. Remove input focus from game
2. Send `PLAYOS_LIFECYCLE_BACKGROUND` through lifecycle pipe to game
3. Transition to `PLAYOS_UI_FOREGROUND_WITH_GAME_BACKGROUND`
4. Map the overlay surface above the game

### `playos-overlay` — System Overlay Client

Create a minimal trusted Raylib overlay client.

**Initial overlay content:**
- Game title and running time
- "Resume Game" option (A button) — sends resume to compositor
- "Quit Game" option — sends `TerminateGame` IPC to `playos-init`
- Battery percentage and thermal status

**Overlay surface policy:**
- The overlay is always pre-spawned but hidden
- Compositor maps it above the game surface when `PLAYOS_BUTTON_SYSTEM` is pressed
- The overlay uses a private Wayland protocol role: `playos_overlay_v1`

**Scene z-order:**
```
1. active game surface
2. dimming layer (semi-transparent black)
3. overlay surface
4. notifications (future)
```

### `playos-platform-api` — Lifecycle Events (Real Implementation)

Replace the stub with a real implementation backed by `playos-runtime`.

**Delivery mechanism:** `playos-init` passes a file descriptor (`PLAYOS_LIFECYCLE_FD`) to the game at launch. The runtime reads from this fd and delivers events to `playos_lifecycle_poll()`.

```c
/* Non-blocking. Returns 1 if event available, 0 if not, -1 on error. */
int playos_lifecycle_poll(PlayOSLifecycleEvent *event);
```

**Game cooperative behavior on `PLAYOS_LIFECYCLE_BACKGROUND`:**
- Pause gameplay
- Stop or mute audio
- Reduce rendering (0 FPS acceptable while backgrounded)
- Write autosave if implemented

**Non-cooperative fallback:** `playos-init` sends `SIGSTOP` to the game process if it does not reduce CPU to near-zero within a timeout (configurable, default: 500ms). `SIGCONT` resumes it. The compositor requests this action; it does not signal processes directly.

### Private Wayland Protocol (Sprint 7 additions)

Extend `playos-runtime/protocols/playos-v1.xml`:

```xml
<interface name="playos_compositor_control_v1" version="1">
  <!-- Sent by trusted runtime to compositor -->
  <request name="set_expected_game">
    <arg name="launch_token" type="string"/>
  </request>
  <request name="terminate_game"/>
  <request name="show_overlay"/>
  <request name="hide_overlay"/>
</interface>

<interface name="playos_overlay_v1" version="1">
  <!-- Identify this client as the trusted overlay -->
  <request name="register_as_overlay"/>
  <!-- Compositor notifies overlay it is about to be shown -->
  <event name="about_to_show"/>
  <event name="about_to_hide"/>
</interface>
```

### Game Exit and Crash Recovery

**Clean exit:**
1. `playos-init` records exit status
2. Compositor destroys or ignores the stale game surface
3. Compositor transitions to `SHELL_FOREGROUND`; unhides and refocuses shell
4. Shell restores previous library position

**Crash (non-zero exit / signal):**
1. Same compositor recovery flow
2. Shell shows a non-intrusive notification: "Game exited unexpectedly"
3. Option: "Restart" (re-sends `LaunchGame`) or "Back to library"

**Invariant:** A game crash must never:
- Reveal a Linux terminal
- Leave the display black for more than 500ms
- Require user to reboot

---

## Acceptance Criteria

- [ ] Selecting a game in the shell launches it; first-frame switch happens (no black flash)
- [ ] Game receives `PLAYOS_LIFECYCLE_FOREGROUND` event at launch
- [ ] System button press shows overlay above the game (dim + overlay visible)
- [ ] Game receives `PLAYOS_LIFECYCLE_BACKGROUND` on System button
- [ ] Overlay "Resume" returns to game; `PLAYOS_LIFECYCLE_FOREGROUND` delivered
- [ ] Overlay "Quit" terminates the game cleanly; shell is shown
- [ ] Clean game exit returns to shell in ≤500ms
- [ ] Simulated crash (`kill -9 <game_pid>`): display returns to shell in ≤500ms, no black screen
- [ ] Second game cannot launch while first is running (one-game rule enforced)
- [ ] `PLAYOS_BUTTON_SYSTEM` key events never appear in the game's input stream
- [ ] Non-cooperative game receives `SIGSTOP`/`SIGCONT` as fallback
- [ ] Compositor state transitions are logged
- [ ] `com.playos.sample-input` game can be launched, used, and exited via the full flow
- [ ] CI passes

---

## Repositories Primarily Involved

| Repo | Work |
|---|---|
| `playos-compositor` | Full state machine, first-frame switching, reserved button, overlay z-order |
| `playos-shell` | Launch request, launcher state UI, crash notification, resume after background |
| `playos-overlay` | Minimal overlay client (resume, quit, battery, thermal) |
| `playos-platform-api` | Real lifecycle event delivery via fd/pipe |
| `playos-runtime` | Lifecycle transport, `set_expected_game` IPC, private Wayland protocol extensions |
| `playos-refdistro` | `playos-overlay` Buildroot package |
| `playos-spec` | Lifecycle event spec, state machine diagram update |

---

## Testing Approach

- Physical ROG Ally for all lifecycle tests
- Lifecycle test matrix: launch → background → resume → quit (×3 cycles)
- Crash test: `kill -9` the game mid-render; verify display recovery time and no terminal
- Non-cooperative test: game that ignores lifecycle events; verify `SIGSTOP` fallback fires
- One-game rule: attempt concurrent `LaunchGame` calls; verify second is rejected
- QEMU: compositor state machine unit tests (no physical GPU required)

---

## Exit Gate

The complete console lifecycle works on the ROG Ally: launch, System button overlay, resume, clean exit, and crash recovery all behave correctly. The player never sees a Linux prompt.

*Previous: [Sprint 6](Sprint-6.md) | Next: [Sprint 8](Sprint-8.md)*
