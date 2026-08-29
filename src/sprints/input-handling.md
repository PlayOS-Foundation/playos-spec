# ROG Ally Input Handling — Architecture Audit & Responsiveness Assessment

**Status:** Architecture audit. The prioritised enhancements (§8 items 1–4, 6) are now implemented in `playos-shell` and `playos-platform-api`; item 5 (reserved-key semantics) still needs a product decision. See the "Implemented" markers in §7/§8.

**Goal:** Document the end-to-end input pipeline, how input reaches the shell, and what needs to change to make the shell feel responsive "to the bone". This supersedes the Sprint 9 retest findings; those are preserved verbatim in the [Appendix](#appendix-sprint-9-retest-findings) at the bottom.

---

## 1. Input pipeline overview

Input travels through **two parallel paths** from the kernel to the UI. They are deliberately separate because they serve different consumers:

```
  Linux kernel evdev
  /dev/input/eventN  (opened O_RDONLY | O_NONBLOCK | O_CLOEXEC)
        │
        ├──▶ (A) GAME PATH — platform-api backend
        │       backend_evdev.c opens 3 fds:
        │         • gamepad  (EV_ABS + EV_KEY + ABS_X/Y/RX/RY + BTN_SOUTH)
        │         • home     (BTN_MODE, no BTN_SOUTH)
        │         • vendor   (KEY_PROG1/PROG2 or BTN_TRIGGER_HAPPY1/2)
        │       → playos_input_get_controller_state()
        │         strips SYSTEM|QUICK_MENU|POWER from the snapshot
        │
        │       → Raylib rcore_playos.c PlayOSPollGamepad()
        │         fills CORE.Input.Gamepad.ready[0]
        │         omits reserved buttons, maps sticks 1:1,
        │         converts triggers [0,1] → [-1,1]
        │
        │       → Raylib EndDrawing() → PollInputEvents()
        │         (refreshes the above at end of each rendered frame)
        │
        └──▶ (B) SHELL PATH — direct evdev (trusted)
                input.c opens the gamepad fd PLUS every reserved node
                (home / vendor / power) and decodes all of it itself.
                Reserved buttons are KEPT here, never given to games.

  Games read only path (A). The shell reads path (B) for buttons, reserved
  keys, AND stick/trigger axes — all decoded from evdev on one fresh frame
  (see §6). The previous Raylib axes overlay was removed to eliminate the
  one-frame stick lag.
```

The key contract point is `playos-platform-api/include/playos/playos_input.h`:

- `playos_button_mask_t` bitmask: SOUTH(A) `1<<0`, EAST(B) `1<<1`, WEST(X) `1<<2`, NORTH(Y) `1<<3`, START `1<<4`, SELECT `1<<5`, SYSTEM `1<<6` (reserved), QUICK_MENU `1<<7` (reserved), DPAD_UP/DOWN/LEFT/RIGHT `1<<8..11`, L1 `1<<12`, R1 `1<<13`, L3 `1<<14`, R3 `1<<15`, POWER `1<<16` (reserved).
- `PlayOSAxis` enum: LEFT_X=0, LEFT_Y=1, RIGHT_X=2, RIGHT_Y=3, LEFT_TRIGGER=4, RIGHT_TRIGGER=5, COUNT=6. Y up is negative; sticks are `[-1,1]`, triggers are `[0,1]`.
- `PlayOSControllerState { buttons; float axes[6]; uint64_t timestamp_us; }`.
- `playos_input_controller_connected()` and `playos_input_get_controller_state()` (returns `0` on success, `-1` when no controller).

---

## 2. Layer 1 — evdev device topology

On the ROG Ally, input is spread across several `/dev/input/event*` nodes. The gamepad node carries the standard controller controls; reserved keys (Home / Command Center / volume / M1-M2) and the hardware Power/Sleep buttons arrive on separate nodes.

The shell's discovery is in `playos-shell/src/input.c`:

| Function | Role |
|---|---|
| `is_gamepad_device()` (`input.c:56`) | Requires all four stick axes (ABS_X/Y/RX/RY) plus BTN_SOUTH. Intentionally does **not** require d-pad capability — hid-asus reports d-pad events without advertising the bits. |
| `find_gamepad_device()` (`input.c:92`) | Scans `/dev/input/event*`, prefers names containing Xbox/X-Box/Microsoft/ASUE/ASUS/ROG Ally/Gamepad. |
| `is_reserved_home_device()` (`input.c:178`) | BTN_MODE present, BTN_SOUTH absent. |
| `is_reserved_vendor_device()` (`input.c:189`) | KEY_VOLUMEUP/DOWN, KEY_PROG1/2 or BTN_TRIGGER_HAPPY1/2 present, and neither BTN_SOUTH nor BTN_MODE. |
| `is_reserved_power_device()` (`input.c:213`) | KEY_POWER or KEY_SLEEP present, and no BTN_SOUTH/BTN_MODE. |
| `shell_input_open_reserved_nodes()` (`input.c:251`) | One full scan; opens every home/vendor/power node exactly once (this is the fix for the historical duplicate-volume-node bug). |

Platform API performs its own independent discovery in `playos-platform-api/src/backends/backend_evdev.c` with the same gamepad capability test, plus separate home and vendor fds.

---

## 3. Layer 2 — platform-api backend (the game path)

File: `playos-platform-api/src/backends/backend_evdev.c`.

- Opens three fds, all `O_RDONLY | O_NONBLOCK | O_CLOEXEC`:
  - `evdev_fd` — the gamepad.
  - `home_fd` — BTN_MODE without BTN_SOUTH (Home).
  - `vendor_fd` — KEY_PROG1/PROG2 or BTN_TRIGGER_HAPPY1/2 (Command Center / Armoury).
- `BUTTON_MAP[]` maps BTN_SOUTH/EAST/WEST/NORTH/START/SELECT/MODE/THUMBL/THUMBR/TL/TR plus KEY_PROG1→SYSTEM, KEY_PROG2→QUICK_MENU, BTN_TRIGGER_HAPPY1→SYSTEM, BTN_TRIGGER_HAPPY2→QUICK_MENU. D-pad is **not** in the button map; it is handled only via ABS_HAT0X/Y.
- `AXIS_MAP[]`: ABS_X/Y/RX/RY → sticks; ABS_Z→LEFT_TRIGGER; ABS_RZ→RIGHT_TRIGGER.
- `normalize_stick()` centers and applies a **0.05 deadzone with no post-deadzone rescale**.
- `normalize_trigger()` maps `[min,max] → [0,1]`, with max detected via `EVIOCGABS(ABS_Z)` and a 255 fallback.
- `drain_fd()` caps at `MAX_EVENTS_PER_CALL 64` events per non-blocking read.
- Discovery is throttled: `RESCAN_INTERVAL_US 2000000` (2 s) and stale-fd re-scan via `fcntl(fd, F_GETFL)`.

`playos-platform-api/src/playos_input.c` is a thin dispatcher to `backend_evdev_get_controller_state()`, then **masks out `SYSTEM|QUICK_MENU|POWER`** from the state games see. That is the security boundary: games can never observe reserved buttons through the platform API.

### 3.1 ROG Ally face-button quirk (X/Y wired rotated)

The Ally's **built-in** controller enumerates as a standard Xbox 360 pad
(`Microsoft X-Box 360 pad`, `Vendor=045e Product=028e`, `event5`, phys
`usb-0000:09:00.3-2/input0`), but its face buttons are wired rotated:
**physical X reports `BTN_NORTH` and physical Y reports `BTN_WEST`** (A/B are
standard). Without a quirk, games and the shell would see X↔Y swapped on the
Ally while external Xbox pads stay correct.

Both `backend_evdev.c` (game path) and `playos-shell/src/input.c` (shell path)
therefore apply a **NORTH↔WEST swap for the gamepad node only**, keyed on the
device identity (`name` contains `X-Box` **and** `phys` starts with
`usb-0000:09:00.3-2`). External pads on other ports are unaffected.
`PLAYOS_ROG_ALLY_FACE_SWAP=1` forces the swap, `=0` forces it off; unset means
auto-detect. The raw-code diagnostic in Settings → Input Test intentionally
still shows the *raw* evdev code (e.g. `0x133` for a physical X press) — the
decoded A/B/X/Y pills are what reflect the corrected logical buttons.

### 3.2 Gamepad identifier database (Sprint 13.6)

The game path and the trusted shell path both resolve the per-device
button/axis remap from the embedded
**SDL_GameControllerDB** (Linux entries, `gamecontrollerdb.inc`) before falling
back to the Xbox-standard table. The shell vendor-copies `gamepad_db.{c,h}` +
`gamecontrollerdb.inc` so the trusted path stays independent of libplayos.
Matching is by the SDL GUID derived from
`EVIOCGID` (bustype/vendor/product/version); GLFW/SDL evdev enumeration order
is reproduced exactly. When a DB entry matches, it already encodes the correct
X/Y for xpad-style pads (`x` binds the `BTN_NORTH` index, `y` the `BTN_WEST`
index), so the §3.1 quirk is **disabled on the DB path** to avoid a double
swap. The quirk still patches the fallback path. See
[Sprint-13.6.md](Sprint-13.6.md).

---

## 4. Layer 3 — Raylib gamepad backend

File: `playos-shell/external/raylib/src/platforms/rcore_playos.c`.

- `PlayOSPollGamepad()` uses gamepad index `0`, calls `playos_input_get_controller_state(&state)`, and sets `CORE.Input.Gamepad.ready[0]`.
- Buttons are mapped, omitting SYSTEM / QUICK_MENU / POWER (they are already stripped upstream, but the backend omits them defensively).
- Sticks are copied 1:1 into Raylib's `axisState[LEFT_X/Y/RIGHT_X/RIGHT_Y]`.
- Triggers are converted `state.axes[..] * 2 - 1` from platform `[0,1]` to Raylib `[-1,1]`.
- `PollInputEvents()` (in `rcore.c`) calls `PlayOSPollGamepad()`, and is invoked from `EndDrawing()` at the **end** of each frame when `SUPPORT_CUSTOM_FRAME_CONTROL` is not defined.

Consequence: Raylib's gamepad state is a **frame-behind snapshot** relative to when the shell next reads it (see §6).

---

## 5. Layer 4 — the shell's direct evdev path (trusted)

File: `playos-shell/src/input.c`.

`shell_input_init()` (around `input.c:568`) performs:

1. `/proc/bus/input/devices` dump to the persistent log (`shell_input_dump_proc_devices`, `input.c:326`).
2. Per-node capability dump (`shell_input_dump_capabilities`, `input.c:355`).
3. Installs a non-blocking `inotify` watch on `/dev/input` (`IN_CREATE|IN_DELETE|IN_ATTRIB`) for gamepad hotplug — best-effort, ignored if unavailable.
4. `find_gamepad_device()`.
5. `shell_input_open_reserved_nodes()`.
6. One-time trigger and stick calibration reads.

Per frame, `shell_input_poll()` (`input.c:865`) does:

1. Saves `controller_prev`, resets `buttons_pressed`.
2. Drains any pending `inotify` events; if the gamepad fd is still missing, retries discovery immediately instead of waiting out the throttle.
3. If the gamepad fd is still missing, retries discovery at most once every `SHELL_INPUT_RESCAN_INTERVAL_SECONDS` = **2 s** (`input.c:863`). Reserved nodes are **not** re-scanned — a deliberate fix, because a full `opendir + open + 2× ioctl` scan costs ~0.5 s on the Ally and caused visible hiccups (`input.c:899-906`).
4. Drains the gamepad fd, then every reserved fd (`input.c:911-915`).

`shell_input_drain_fd()` (`input.c:815`) reads events non-blocking and, crucially, **breaks on EV_SYN** — it processes exactly **one kernel input frame per poll**. That is the right latency behaviour: it keeps the shell glued to the most recent frame and never replays stale batched events.

`shell_input_process_event()` (`input.c:646`) decodes:

- EV_KEY face buttons, d-pad (`ABS_HAT0X/Y` and `BTN_DPAD_*` forms), L1/R1, stick clicks.
- **Reserved** keys: SYSTEM / QUICK_MENU / POWER / volume / M1-M2 (these stay in `s->controller.buttons`).
- EV_ABS sticks (ABS_X/Y/RX/RY → `normalize_stick`, 5% deadzone) and triggers (ABS_Z/RZ).

Queries:

- `shell_input_button_pressed()` (`input.c:918`) — edge detect plus an event-level catch so fast taps that resolve within a single poll are not lost.
- `shell_input_button_released()` (`input.c:944`) — falling edge.
- `shell_input_button_held()` (`input.c:955`) — level query ("IsBeingPressed"). The Live Input Test uses this for every pill.

---

## 6. How a 60 Hz frame flows

File: `playos-shell/src/main.c`.

```
SetTargetFPS(60)
loop:
  1. shell_input_poll(s)          → s->controller.buttons AND s->controller.axes
                                    (fresh evdev: buttons, sticks, triggers,
                                     reserved keys — one frame, no lag)
  2. lifecycle / update screen    → uses s->controller
  3. draw screen
  4. render_end_frame() → EndDrawing() → PollInputEvents()
                                     → PlayOSPollGamepad() refreshes Raylib state
  5. s->frame_time = time after EndDrawing()
```

Buttons and sticks now share the same fresh evdev frame in step 1. The previous
`shell_read_gamepad_axes()` Raylib overlay (step 2 in the earlier revision) was
removed because it read the Raylib snapshot produced at the end of the previous
frame, introducing a ~16.7 ms stick lag and a second (10%) deadzone. Raylib's
gamepad state is still refreshed at the end of each frame for any future
Raylib-side consumers, but the shell no longer uses it for its own state.

The FPS HUD in `screen_home.c` now reports `1.0 / frame_time` instead of milliseconds.

---

## 7. Responsiveness assessment

### Strengths

- All evdev fds are non-blocking; nothing can stall the render loop on a blocking read.
- The shell drains **one kernel frame per poll** (`input.c` breaks on EV_SYN) — no stale event replay.
- Platform API caps its own drain at 64 events/call.
- `SetTargetFPS(60)` gives a stable cadence; `EndDrawing()` does the frame pacing.
- Reserved-button reads are on the same fresh per-frame path as game buttons (no frame lag there).
- Missing-device retries are throttled, and the expensive full scan was removed from the hot path.

### Problems / gaps

1. **One-frame stick lag (primary).** ~~Sticks and triggers enter via Raylib, which refreshes at `EndDrawing()` — one frame after the shell reads it (`main.c` ordering).~~ **Implemented:** the Raylib axes overlay was removed; `shell_input_poll()` now decodes sticks/triggers from evdev on the same fresh frame as buttons.

2. **Deadzone inconsistency.** ~~The shell's direct evdev stick decode uses a **5%** deadzone, while the Raylib overlay uses **10%**.~~ **Implemented:** the overlay is gone and the single `SHELL_STICK_DEADZONE 0.05f` constant in `input.c` governs all shell stick decoding, matching the platform-api backend.

3. **Two independent opens of the same nodes.** Platform API and the shell each open their own fds and drain the gamepad independently. This is by design (trust boundary), but it means duplicate reads and a risk of divergence if the two decoders ever disagree. Documenting it as intentional is fine; keeping the two decoders byte-for-byte compatible is the real cost.

4. **Hotplug latency up to 2 s.** ~~Both platform API and shell throttle missing-device re-scan to 2 s.~~ **Implemented for the shell:** an `inotify` watch on `/dev/input` triggers an immediate gamepad re-scan on CREATE/DELETE, while the 2 s throttle remains as a fallback. The platform-api backend still throttles at 2 s (acceptable for game processes).

5. **Reserved keys are momentary pulses.** Home/Command/M1-M2 arrive as 7–9 ms `value=1` → `value=0` pulses with no autorepeat (historical Finding 3). The shell already latches them visually; semantic actions must be edge-triggered, never level-triggered, for those codes. **Still open — needs a product decision on the intended Armoury/Command semantics.**

6. **No startup "controller present" snapshot for reserved nodes.** ~~The capability dump exists… the remaining gap is ensuring the *game-facing* backend also logs its chosen fds at discovery time for cross-correlation.~~ **Implemented:** `open_controller()` in `backend_evdev.c` now logs the chosen gamepad fd and device name (both preferred and fallback paths), alongside the existing home/vendor fd logs.

7. **Timestamping is not surfaced.** ~~`PlayOSControllerState.timestamp_us` exists but the shell's own path does not timestamp frames.~~ **Implemented (opt-in):** `shell_input_drain_fd()` measures queue-to-drain age using `clock_gettime(CLOCK_MONOTONIC)` vs the kernel `input_event.time`, logged once per second when `PLAYOS_INPUT_LATENCY_LOG` is set. Off by default to avoid per-event overhead on the 60 Hz hot path.

---

## 8. Recommended enhancements (prioritised)

1. **De-lag the shell stick path.** ~~Read Raylib axes after `EndDrawing()`/`PollInputEvents()`, or read sticks from evdev directly…~~ **Implemented:** sticks are now read from evdev directly in `shell_input_poll()`; the Raylib overlay was removed.

2. **Unify deadzone handling.** ~~One constant, one function, used by both the shell evdev decode and the Raylib overlay…~~ **Implemented:** single `SHELL_STICK_DEADZONE 0.05f` constant; the Raylib overlay no longer exists to disagree.

3. **Measure end-to-end input latency.** ~~Add monotonic timestamps at evdev drain, platform snapshot, Raylib poll, and frame start; log or expose them behind a debug flag.~~ **Implemented (partial, opt-in):** evdev queue-to-drain age is measured and logged once per second when `PLAYOS_INPUT_LATENCY_LOG` is set. Full cross-layer (platform snapshot → Raylib → frame start) instrumentation is still a future extension if deeper profiling is needed.

4. **Replace the 2 s gamepad re-scan with inotify** on `/dev/input` (or at least shorten the throttle after a successful boot). ~~Keep reserved-node discovery one-time as today.~~ **Implemented:** inotify immediate re-scan for the gamepad; reserved-node discovery remains one-time.

5. **Reserved-key semantic handling.** ~~Confirm the intended Armoury/Command semantics for the currently-unmapped F15/F17 codes and wire them as edge-triggered actions; never depend on a held state for pulse-only codes.~~ **Still open — needs a product decision.** No default mappings were invented.

6. **Cross-correlate both decoders.** ~~Log the platform-api backend's chosen gamepad/home/vendor fds at startup next to the shell's dump…~~ **Implemented:** `open_controller()` logs the gamepad fd + name (preferred and fallback); home/vendor fds were already logged.

7. **Keep one kernel frame per poll.** This is already correct; protect it in any refactor. Do not "drain all" in the shell hot path — that reintroduces stale-event latency. **Unchanged and preserved.**

---

## Appendix — Sprint 9 retest findings

**Historical context.** These were the findings from the Ally USB image retest after the volume-node discovery + HOME/COMMAND latch changes. Several are now fixed (volume node, power button); the pulse-behaviour notes remain relevant.

### Discovered input topology (from shell logs, at the time)

- Gamepad: `Microsoft X-Box 360 pad` at `/dev/input/event5`.
- Vendor node: `Asus Keyboard` at `/dev/input/event8` (fd=9).
- Three `Asus Keyboard` nodes skipped by the gamepad matcher as "missing stick axes": `event6`, `event7`, `event8`.
- The volume-node matcher selected **`event8` again** (fd=10) — a duplicate of the vendor node.

### Finding 1 — Volume node discovery found a duplicate, not the real volume node

```
found vendor node: 'Asus Keyboard' (/dev/input/event8) fd=9
scanning /dev/input/event* for volume node...
found volume node: 'Asus Keyboard' (/dev/input/event8) fd=10
```

`event8` was opened twice and drained twice; `event6`/`event7` were never opened. **Fixed** by `shell_input_open_reserved_nodes()` opening every Asus/home/vendor/power node exactly once.

### Finding 2 — Volume Up / Volume Down never reach the shell

No `KEY_VOLUMEUP` (`0x72`) / `KEY_VOLUMEDOWN` (`0x73`) events were observed. Only PROG1 (`0x94`), F15 (`0xb9`), F16 (`0xba`), F17 (`0xbb`), F18 (`0xbc`) on `event8`. **Fixed** — volume keys now arrive and light the Live Input Test.

### Finding 3 — HOME and COMMAND are momentary pulses, not held keys

Every `0x94` (HOME) and `0xba` (COMMAND) press is a 7–9 ms `value=1`→`value=0` pulse with no autorepeat. Same for F17/F18. The shell's 0.6 s visual latch is the correct accommodation; semantic actions must be edge-triggered.

### Finding 4 — F15 behaves differently (level + autorepeat)

`0xb9` (F15) emits `value=1`, then `value=2` repeats (~260 ms, then ~40 ms), then `value=0`. F15 is a real held key, unlike the pulse keys.

### Finding 5 — Persistent logs did not enumerate input devices

`init.log` had no `/proc/bus/input/devices` dump. **Fixed** — `shell_input_dump_proc_devices()` now logs it, plus `shell_input_dump_capabilities()` logs per-node names/phys/interesting evbits.

### Finding 6 — shell-stderr.log contained two boot cycles

The persistent log is append-across-reboots (expected). Findings were read against the second boot's ~41 s mark.

### Finding 7 — Power button detection implemented (Sprint 9 follow-up)

`KEY_POWER` (`0x74`) and `KEY_SLEEP` (`0x8e`/142) on the ACPI Power/Sleep nodes are now decoded into `PLAYOS_BUTTON_POWER` (bit 16, reserved — games never see it). The shell discovers the power node via `is_reserved_power_device()`, opens it without `EVIOCGRAB` so kernel ACPI handling is unaffected, and shows a `PWR` pill in the Live Input Test with the 0.6 s pulse latch. `shell_input_button_held()` was added and the Live Input Test now uses it for every pill. **Status:** implemented; compiles and builds, awaiting Ally hardware retest.
