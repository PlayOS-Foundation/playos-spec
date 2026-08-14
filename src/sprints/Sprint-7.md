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
- **Compositor control transport:** Unix socket `/run/playos/compositor.sock` only — the private Wayland `playos_compositor_control_v1` interface is removed from this sprint; `playos_overlay_v1` remains for the overlay client
- **Process ownership:** `playos-init` spawns and supervises `playos-compositor`, `playos-shell`, and `playos-overlay`; the compositor maps/unmaps surfaces but never spawns processes

---

## Decision Locked: Compositor Control Transport

`playos-init` / `playos-runtime` control the compositor exclusively over the Unix socket **`/run/playos/compositor.sock`** (`SOCK_SEQPACKET`, `root:playos-trusted`, `0660`), as specified in [`runtime-ipc.md`](../runtime-ipc.md) §2 and §7. Messages: `SetExpectedGame`, `ClearExpectedGame`, `ForceTerminateGame`, `ShowOverlay`, `HideOverlay` (init → compositor) and `GameSurfaceReady`, `CompositorStateChanged` (compositor → init). [`playos-init-spec.md`](../playos-init-spec.md) binds this socket at boot and routes the non-cooperative `SIGSTOP`/`SIGCONT` request through it.

The private Wayland interface **`playos_compositor_control_v1` is removed** from this sprint — it duplicated the socket's control surface over the Wayland wire. **`playos_overlay_v1` is kept**: the overlay is a Wayland client, so its role registration (`playos_manager_v1::register_overlay`) and its `set_surface` / `surface_ready` / `request_dismiss` requests plus `about_to_show` / `about_to_hide` / `output_info` events belong on the Wayland protocol.

Follow-up reconciliation:

- Remove the Wayland interface `playos_game_launch_v1` from `playos-v1.xml` (currently declared in `playos-runtime`, `playos-compositor`, and vendored copies). It duplicates the socket's `SetExpectedGame` / `ClearExpectedGame` / `GameSurfaceReady` control surface, and is currently only declared in the XML — not implemented server-side (`playos-compositor/README.md`). The compositor must drive this path over `/run/playos/compositor.sock` using the `playos_ipc_*` framing library in `playos-init/ipc/`.
- `playos-compositor` does not yet connect to `/run/playos/compositor.sock` (its readiness handshake is currently a file). Add the socket client and retire the Wayland game-launch interface this sprint.

---

## Scope

### In Scope

- Full game launch flow (shell → init → compositor → game)
- Complete compositor state machine
- Reserved system button intercept
- `playos-overlay` trusted client (minimal: resume, quit, battery, thermal)
- Real lifecycle event delivery via `PLAYOS_LIFECYCLE_FD`
- Game exit (clean and crash) with shell recovery
- Private Wayland protocol Sprint 7 changes (overlay retained; game-launch removed)
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
| `playos-compositor` | Full state machine, first-frame rule, system button intercept, overlay surface management, compositor socket client |
| `playos-shell` | Launch IPC, launching-state spinner UI, crash notification, library restore |
| `playos-runtime` | Lifecycle transport, compositor control IPC client (`SetExpectedGame`), `playos_overlay_v1` Wayland extension |
| `playos-platform-api` | Real lifecycle event delivery (fd-backed) |
| `playos-refdistro` | `playos-overlay` trusted Raylib client + Buildroot package |
| `playos-spec` | Lifecycle event spec, state machine diagram update |

---

## Expected Files and Directories

### `playos-compositor`

```text
src/compositor_ipc.c       # client for /run/playos/compositor.sock
src/state_machine.c        # all states and transitions
src/system_button.c        # input intercept at seat level
src/overlay_manager.c      # z-order, show/hide logic
```

### `playos-runtime`

```text
protocols/playos-v1.xml    # Sprint 7: keep playos_overlay_v1, remove playos_game_launch_v1
src/lifecycle_transport.c  # fd creation, event write
```

### `playos-refdistro`

```text
br2-external/package/playos-overlay/
    playos-overlay.mk
    Config.in
src/playos-overlay/
    main.c                 # Raylib overlay UI
```

---

## Agent Task Breakdown

### Task Status Grid

| Task ID | Task | Primary repo | Status | Notes / evidence |
|---|---|---|---|---|
| S7-T1 | Implement full game launch flow in `playos-init` | `playos-init` | not started | |
| S7-T2 | Implement full compositor state machine | `playos-compositor` | not started | |
| S7-T3 | Implement system button intercept at seat level | `playos-compositor` | not started | |
| S7-T4 | Build `playos-overlay` trusted Raylib client | `playos-refdistro` | not started | |
| S7-T5 | Implement lifecycle fd delivery in platform-api | `playos-platform-api` | not started | |
| S7-T6 | Finalize private Wayland protocol (overlay kept, game-launch removed) | `playos-runtime` | not started | |
| S7-T7 | Implement game exit and crash recovery | `playos-compositor`, `playos-refdistro` | not started | |
| S7-T8 | Integration and lifecycle validation on Ally | `playos-refdistro` | not started | |

### S7-T1 — Implement full game launch flow in `playos-init`

1. Validate: one-game rule; reject immediately if a game is already running
2. Validate: manifest exists, executable exists, `api_version` ≤ current system version
3. Prepare environment variables: `PLAYOS_GAME_ID`, `PLAYOS_INSTALL_PATH`, `PLAYOS_SAVE_PATH`, `PLAYOS_CACHE_PATH`, `WAYLAND_DISPLAY=playos-0`, `PLAYOS_LIFECYCLE_FD`, `PLAYOS_LAUNCH_TOKEN`
4. Create lifecycle pipe; store write-end in `playos-init`, pass read-end as `PLAYOS_LIFECYCLE_FD`
5. Emit `SetExpectedGame { launch_token, game_id }` to the compositor over `/run/playos/compositor.sock`
6. Spawn game executable; track PID
7. Emit `GameStarted { game_id, pid, launch_token }` to the shell over `control.sock`

> **Note:** the `GameStarted` / `GameExited` / `GameCrashed` message types are already declared in `ipc/ipc.h`, but `playos-init` currently **never emits them** — `playos_supervisor_game_exited` only writes to `init.log`. S7-T1 wires up `GameStarted`; S7-T7 wires up the exit/crash emissions. Do not add new protocol types.

**Done when:** `com.playos.sample-input` launches from the shell, receives the environment variables, the compositor logs `SetExpectedGame`, and the shell receives `GameStarted`.

### S7-T2 — Implement full compositor state machine

Implement all five states and all transitions:

```
SHELL_FOREGROUND
  → (SetExpectedGame received)    → GAME_STARTING
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
- When in `GAME_FOREGROUND`: remove input focus from the game, emit `CompositorStateChanged` (`PLAYOS_UI_FOREGROUND_WITH_GAME_BACKGROUND`) to `playos-init`, and transition to the overlay state; `playos-init` then writes `PLAYOS_LIFECYCLE_BACKGROUND` to the lifecycle fd and arms the non-cooperative `SIGSTOP` timer
- Verify via `evtest` and a game that logs all key events — `PLAYOS_BUTTON_SYSTEM` must never appear in the game log

**Done when:** pressing the system button in-game activates the overlay and the game never sees the key event.

### S7-T4 — Build `playos-overlay` trusted Raylib client

Initial overlay content:
- Game title and elapsed running time
- "Resume Game" (A button) — sends `playos_overlay_v1::request_dismiss` to the compositor (the compositor hides the overlay and returns focus to the game)
- "Quit Game" (B or menu) — sends `TerminateGame` IPC to `playos-init`
- Battery percentage (stub value acceptable this sprint)
- Thermal status (stub value acceptable this sprint)

Surface policy:
- Pre-spawned at boot, hidden; compositor shows/hides it via `playos_overlay_v1`
- Overlay registers as the trusted overlay via `playos_manager_v1::register_overlay`
- Overlay implements `playos_overlay_v1::set_surface` / `surface_ready` / `request_dismiss` and handles `about_to_show` / `about_to_hide` to reset state cleanly

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

### S7-T6 — Finalize private Wayland protocol (overlay kept, game-launch removed)

Compositor control stays on `/run/playos/compositor.sock` (see the decision locked above); no Wayland control interface is added this sprint.

`playos-overlay` remains a Wayland client, so `playos_overlay_v1` is retained with its correct interface shape in `playos-runtime/protocols/playos-v1.xml`:

```xml
<interface name="playos_overlay_v1" version="1">
  <request name="set_surface">
    <arg name="surface" type="object" interface="wl_surface"/>
  </request>
  <request name="surface_ready"/>
  <request name="request_dismiss"/>
  <event name="about_to_show"/>
  <event name="about_to_hide"/>
  <event name="output_info">
    <arg name="width"        type="int"/>
    <arg name="height"       type="int"/>
    <arg name="refresh_mhz"  type="uint"/>
    <arg name="scale_100"    type="uint"/>
  </event>
</interface>
```

The overlay's trusted role is registered through `playos_manager_v1::register_overlay`, not a `register_as_overlay` request on the overlay interface.

Remove the `playos_game_launch_v1` interface from the XML. Its four operations — `set_expected_game`, `clear_expected_game`, `game_surface_ready`, `game_surface_destroyed` — duplicate the socket's three messages `SetExpectedGame` / `ClearExpectedGame` / `GameSurfaceReady`, so this path belongs on `/run/playos/compositor.sock`, not the Wayland wire.

Regenerate Wayland scanner outputs. Only the trusted `playos-overlay` client uses `playos_overlay_v1`.

**Done when:** compositor binds `playos_overlay_v1`; overlay registers via `playos_manager_v1::register_overlay`; `playos_game_launch_v1` is gone from the protocol; CI protocol scanner passes.

### S7-T7 — Implement game exit and crash recovery

**Clean exit:**
1. `playos-init` records exit status; closes lifecycle pipe write-end
2. `playos-init` emits `GameExited { game_id, exit_code }` to the shell over `control.sock` (types already declared in `ipc/ipc.h` — no new protocol)
3. Compositor detects Wayland client disconnect; transitions to `SHELL_FOREGROUND`
4. Shell surface is unhidden and refocused; library scroll position restored
5. Shell shows no notification on clean exit

**Crash (non-zero exit or signal):**
1. Same compositor recovery
2. `playos-init` emits `GameCrashed { game_id, exit_code, signal }` to the shell over `control.sock`
3. Shell shows a non-intrusive toast notification: "Game exited unexpectedly"
4. Options: "Restart" (re-sends `LaunchGame`) or "Back to Library" (dismiss)

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

Compositor state-machine transitions are additionally covered by QEMU unit tests (no physical GPU required).

**Done when:** all 7 test cases pass on the Ally with evidence logged.

---

## Implementation Guidance

### Launch token

The launch token is a one-time random UUID generated per launch. It expires when the first Wayland client presents with that token. Clients presenting an unknown or expired token during the launch window are rejected and logged, not crashed.

### Display switch timing

The shell surface must remain visible until the game's first Wayland buffer is committed. Do not hide the shell on `GameStarted` — hide it only on the first-frame transition.

### Overlay pre-spawn

`playos-overlay` is pre-spawned by `playos-init` at boot (like `playos-shell`) and supervised with restart-on-exit. The compositor maps/unmaps its surface; it never spawns or kills the overlay process. This avoids visible latency when the system button is pressed.

### Game cooperative behavior on background

On `PLAYOS_LIFECYCLE_BACKGROUND`, a cooperative game should pause gameplay, stop/mute audio, reduce rendering (0 FPS acceptable while backgrounded), and write an autosave if implemented. If CPU does not drop within `GAME_PAUSE_TIMEOUT_MS` (default 500ms), `playos-init` falls back to `SIGSTOP`.

### Code hygiene (deferred cleanup)

While touching `playos-init` for S7-T1/S7-T7, remove the dead code that Sprint 6 left behind so the launch/lifecycle path has a single obvious implementation:

- `playos_spawn_child()` in `src/child_process.c` — unused; the supervisor spawns everything directly.
- `playos_supervisor_spawn_test_client()` / `spawn_test_client()` in `src/supervisor.c` — unused; the `ipc-test-client` self-test is spawned from `main.c`.
- `playos_recovery_enter()` / `playos_recovery_loop()` in `src/recovery.c` — unused; the live recovery path is `playos_enter_recovery()` in `src/supervisor.c`. Delete `recovery.c` (and its header) or fold the banner into `playos_enter_recovery()`.

**Done when:** `grep` finds no callers of the above and the tree still builds/tests green.

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
- [ ] Compositor state transitions are logged
- [ ] Full 3-cycle lifecycle test passes without state leaks
- [ ] CI passes

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
