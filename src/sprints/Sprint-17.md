# Sprint 17 — Touch Input + On-Screen Keyboard (OSK)

**Goal:** Deliver touch input to PlayOS applications — through the platform API that already carries the controller, not through the compositor (see [ADR-0013](../adr/ADR-0013-input-delivery.md)) — and ship a reusable on-screen keyboard that any foreground client can invoke and receive committed text from. The compositor's seat forwarding is retained for *foreign* Wayland clients, which do not exist yet.

**Primary Outcome:** A finger tap on the ROG Ally touchscreen reaches the focused surface as a raylib `GetTouchPosition()` point, and a system OSK can be raised from either a shell text field (e.g. Wi-Fi passphrase) or a game text field, delivering the typed string back to the invoking client via a standard text-input protocol.

**Status:** 🟡 Post-MVP — **re-scoped by ADR-0013 (2026-09-26) following confirmed product direction**: every game and app is SDK-built, so there are no foreign Wayland clients and path 2 is not planned. What this sprint actually needs: (a) touch in `playos-platform-api`'s evdev backend (the kernel half is already done and verified on hardware); (b) an OSK rendered by the **overlay** and driven by a PlayOS text-entry API; (c) the shell/app consumers. The seat forwarding written in T1 is retained as the implementation for a future client class but is inert today, and `zwp_text_input_v3`/T2/T4 are not on the product's path.

**Prerequisites:** MVP complete (Sprint 15); Sprint 16 networking (the Wi-Fi passphrase field is the shell's first real text-input consumer); the `rcore_playos.c` gamepad translation landed (raylib `CORE.Input.Gamepad.*` fed from `playos_input_get_controller_state`).

---

## Why This Sprint Exists

The MVP shell is navigated entirely by D-pad and face buttons; there is no text entry and no pointer/touch. Every Tier-1 text-entry feature — Wi-Fi passphrase (Sprint 16), search, save-file naming, user profiles — needs a keyboard, and the ROG Ally's 7″ touchscreen is currently inert (nothing forwards `wl_touch`). This sprint delivers both: touch becomes a real input source, and an OSK provides text entry.

Crucially, the OSK is **not** a shell-only widget. It is a system service that a game can invoke too, so any raylib game gains on-screen text input without shipping its own keyboard UI. This is the same model consoles use (Steam Deck's OSK is compositor-level, not per-game).

---

## Start Condition Checklist

- MVP verified on hardware; **Sprint 16 is done**. Its Wi-Fi passphrase field is a working text-entry consumer — but with a hand-rolled keyboard inside `screen_network.c` (CAPS / 123-symbols / SHOW·HIDE). Sprint 17 replaces that keyboard with the system OSK; the field, the masking and the commit logic already exist and work.
- **The Ally has no touchscreen driver in the kernel config today.** `board/ally/linux.config` enables `INPUT_EVDEV`, `INPUT_JOYSTICK`, `HID_GENERIC`/`HID_ASUS`/`HID_PLAYSTATION`/`HID_XBOX` and the Designware I2C controller, but **no** `TOUCHSCREEN_*`, no `I2C_HID`, no `HID_MULTITOUCH` (checked 2026-09-25). Touch cannot work until that stack is enabled — this is the first half of T1.
- Sprint 8 gamepad wiring merged: `rcore_playos.c` translates `playos_input_get_controller_state()` into `CORE.Input.Gamepad.*`.
- Sprint 7 overlay architecture live: `playos-overlay` is a separate trusted raylib process that maps above any surface (`playos_overlay_v1`), owns "Virtual keyboard (future)" per `playos-overlay-spec.md`.
- Compositor uses `wlr_scene` (shell/game/overlay trees) + `wlr_seat "seat0"`, but forwards **no** pointer/touch today (`system_button.c` intercepts keyboard `BTN_MODE` only).
- wlroots 0.20 pinned (provides `wlr_text_input_v3` and `wlr_seat_touch_notify_*`). Verified: `output/ally/build/wlroots-0.20.0/include/wlr/types/` carries both `wlr_text_input_v3.h` and `wlr_cursor.h`.

---

## Decisions Locked for This Sprint

- **Touch delivery depends on the client class (ADR-0013).** First-party clients (shell, overlay, SDK games) get touch through `playos-platform-api`'s evdev backend, like the gamepad. Foreign Wayland clients get it through the seat: `wl_touch`, hit-tested in the scene.
- **Pointer via `wl_pointer`** alongside touch (the same seat plumbing); this also makes raylib `GetMousePosition()`/`IsMouseButton*()` work for USB mice and touch-as-mouse.
- **Text input: an explicit PlayOS API for first-party clients**, driven over `playos_overlay_v1` (T5/T6). `zwp_text_input_v3` remains the route for foreign Wayland clients, and is worth implementing when those arrive — not before, since nothing on the current path can bind it.
- **The OSK UI lives in `playos-overlay`** (already owns "Virtual keyboard (future)"), rendered as a raylib component. It is *one* system keyboard, not a per-game widget.
- **OSK visibility is compositor-driven:** when the focused client *enables* text input, the compositor signals the overlay to show the OSK; on disable/hide it unmaps. The game does not render or size the OSK.
- **Committed text flows compositor → focused client** via `zwp_text_input_v3::commit_string`. Text never crosses `control.sock`; the OSK only produces Wayland protocol events.
- **Overlay ↔ compositor OSK coordination** is a small additive extension to `playos_overlay_v1` (see S17-T6). The overlay renders and hit-tests keys; the compositor is the only party that talks to the focused client.
- **Game-facing API is a raylib extension**, not a new `libplayos` ABI: `rcore_playos.c` owns the game's Wayland connection, so it implements the `zwp_text_input_v3` client and exposes `ShowOnScreenKeyboard()` / `HideOnScreenKeyboard()` plus `GetCharPressed()` (which "just works" for the committing client). Non-raylib engines implement `zwp_text_input_v3` directly.
- **No keyboard input forwarding change in this sprint.** The OSK produces text through the text-input protocol; physical keyboard forwarding (`system_button.c` currently withholds non-reserved keys) remains a separate, later concern.
- **Reserved buttons stay reserved.** The OSK never synthesizes `SYSTEM`/`QUICK_MENU`; text input is a data channel, not an evdev injection path.

---

## Scope

### In Scope

- Compositor pointer + touch seat forwarding (`wlr_cursor`, `wlr_seat_pointer_notify_*`, `wlr_seat_touch_notify_*`, `wlr_scene_node_at` hit-testing, per-output coordinate transform).
- Compositor `zwp_text_input_v3` manager + focus routing to the focused surface.
- Compositor → overlay OSK show/hide signaling, and overlay → compositor key/string commit.
- Raylib backend (`rcore_playos.c`): `wl_touch` + `wl_pointer` → `CORE.Input.Touch.*` / mouse; `zwp_text_input_v3` client → `charPressedQueue`; `ShowOnScreenKeyboard()` / `HideOnScreenKeyboard()` extension.
- Overlay OSK UI: qwerty layout, touch tap-to-type, shift/caps, backspace/enter/space, numeric/password variants.
- Shell text-field integration (Wi-Fi passphrase is the first consumer).
- `com.playos.sample-osk` game that invokes the OSK and echoes typed text.
- `wayland-protocol.md` and `playos-overlay-spec.md` updates.

### Explicitly Out of Scope

- Physical keyboard forwarding to clients (`system_button.c` change) — separate sprint.
- Mouse-only desktop pointer UX beyond what `wl_pointer` gives raylib for free.
- Multi-touch gestures (pinch/zoom), multi-touch beyond `MAX_TOUCH_POINTS` tracking.
- Haptics on keypress, predictive text, autocorrect, IME/composition, CJK input methods.
- Per-game OSK skins/theming (one system theme for now).
- On-screen keyboard over external displays (single built-in panel first).

---

## Required Repository Changes

| Repo | Required work |
|---|---|
| `playos-compositor` | `wlr_cursor` + pointer/touch notify + `wlr_scene` hit-testing; `wlr_text_input_v3` manager + focus routing; OSK show/hide + commit plumbing |
| `playos-shell` (vendored raylib) | `rcore_playos.c` touch/pointer → `CORE.Input.Touch.*`/mouse; `zwp_text_input_v3` client + `ShowOnScreenKeyboard()` extension |
| `playos-overlay` (`playos-refdistro/src/playos-overlay/`) | OSK UI component + layout engine + touch tap-to-type + commit requests |
| `playos-runtime` | `playos-v1.xml` overlay protocol extension (OSK visibility/commit) + regenerated headers |
| `playos-samples` | `com.playos.sample-osk` game |
| `playos-refdistro` | `board/ally/linux.config`: touchscreen input stack (T1, first half). `package/playos-samples/playos-samples.mk`: build/install rules for the new sample — that list is **explicit per sample, it does not glob**, so adding a directory alone ships nothing |
| `playos-spec` | This sprint; `wayland-protocol.md` (text-input + touch sections); `playos-overlay-spec.md` (OSK screen); `post-mvp.md` entry |

---

## Expected Files and Directories

### `playos-compositor`

```text
src/input.c                    # NEW: wlr_cursor, pointer/touch listeners, scene hit-testing
src/text_input.c               # NEW: wlr_text_input_v3 manager + focus routing + commit
src/osk.c                      # NEW: OSK show/hide state, overlay signal, commit bridge
src/system_button.c            # unchanged (keyboard intercept stays as-is)
```

### `playos-shell` (vendored raylib)

```text
external/raylib/src/platforms/rcore_playos.c   # wl_touch/wl_pointer listeners; zwp_text_input_v3 client
external/raylib/src/rcore.c                     # (no change) PollInputEvents already resets touch/pointer state
external/raylib/src/raylib.h                    # ShowOnScreenKeyboard / HideOnScreenKeyboard decls (PlayOS section)
```

### `playos-overlay` (`playos-refdistro/src/playos-overlay/`)

```text
playos-overlay/osk.c            # NEW: OSK screen + layout + tap-to-type
playos-overlay/osk_layouts.c    # NEW: qwerty/numeric/password layout tables
```

The overlay is a single `main.c` plus `CMakeLists.txt` and `protocols/` today — there is **no** `src/` subdirectory, and no `core_keyboard_testbed.c`.

### `playos-refdistro`

```text
br2-external/board/ally/linux.config                      # touchscreen input stack (T1)
br2-external/package/playos-samples/playos-samples.mk      # new sample's build/install rules
```

### `playos-runtime`

```text
protocols/playos-v1.xml         # playos_overlay_v1 OSK requests/events (see S17-T6)
```

### `playos-samples`

```text
osk-demo/src/main.c             # game: text field + ShowOnScreenKeyboard + GetCharPressed echo
osk-demo/manifest.json          # com.playos.sample-osk
```

---

## Agent Task Breakdown

### Task Status Grid

| Task ID | Task | Primary repo | Status | Notes / evidence |
|---|---|---|---|---|
| S17-T1 | Touchscreen input stack **+** pointer/touch seat forwarding | `playos-refdistro`, `playos-compositor` | **compositor half written, device path unverified** | `src/input.c` (wlr_cursor + scene hit-test + pointer/touch notify + seat capabilities POINTER|TOUCH + listener teardown) compiles clean in all three targets and is confirmed running in the guest (`input.c:389: pointer/touch forwarding enabled`). Kernel half done earlier. **Not yet verified:** attach and delivery — the emulator image delivers no input devices (see the CI note), so this needs the hardware flash |
| S17-T2 | `zwp_text_input_v3` manager + focus routing | `playos-compositor` | not started | `wlr_text_input_v3` |
| S17-T3 | Raylib backend touch/pointer → `CORE.Input.Touch.*`/mouse | `playos-shell` | not started | `wl_touch`/`wl_pointer` listeners |
| S17-T4 | Raylib backend text-input client + `ShowOnScreenKeyboard()` | `playos-shell` | not started | `zwp_text_input_v3` client |
| S17-T5 | Overlay OSK UI + layout + tap-to-type | `playos-overlay` | not started | raylib component |
| S17-T6 | Overlay ↔ compositor OSK commit protocol | `playos-runtime` | not started | extend `playos_overlay_v1` |
| S17-T7 | Shell text-field integration (Wi-Fi passphrase) | `playos-shell` | not started | first consumer |
| S17-T8 | OSK sample game + end-to-end validation | `playos-samples` | not started | `com.playos.sample-osk` |

### S17-T1 — Pointer + touch seat forwarding in compositor

**First, the device has to produce touch events at all.** `board/ally/linux.config` carries no touchscreen driver, so this half has the same shape as Sprint 16's T1 did for Wi-Fi: enable the I2C-HID + HID-multitouch stack, rebuild, and confirm the panel shows up as an evdev node with `ABS_MT_*` axes. The panel is an I2C HID device; read its ACPI HID from `dmesg` / `/sys/bus/i2c/devices` **on the hardware** before choosing a driver rather than guessing one.

Then the compositor half — it creates `seat0` but forwards nothing except the keyboard `BTN_MODE` intercept. Add:

- Create a `wlr_cursor` bound to the output layout; create an `wlr_xcursor_manager` (or use `wlr_cursor_set_image`) for the pointer sprite.
- Handle backend pointer events (`wlr_backend.events.new_pointer`, axis/motion/button/frame) and touch events (`new_touch`, down/up/motion/cancel).
- For each pointer/touch position, hit-test with `wlr_scene_node_at(scene, lx, ly, &sx, &sy)` and, when the result is a `wlr_scene_surface`, call `wlr_seat_pointer_notify_enter()` / `wlr_seat_touch_notify_down()` with the surface-local coordinates. Map to the surface via `wlr_scene_surface_from_node()`.
- Apply the correct per-output transform and scale (the output may be rotated/scaled); use `wlr_output_layout_get_at()` and the surface's `current.x`/`current.y`.
- Route touch/pointer only to the **focused** surface (respect the same z-order the overlay already uses: game < shell < overlay). When the OSK is visible, the overlay is top-most and receives the events.

**Done when:** tapping the ROG Ally screen with a debug build logs a `wl_touch` down/up with correct surface-local coordinates, and `wlr_seat_touch_notify_down/up` fire for the top-most surface at that point.

### S17-T2 — `zwp_text_input_v3` manager + focus routing

- Create `wlr_text_input_manager_v3` via `wlr_text_input_manager_v3_create(display)`.
- On `wlr_text_input_v3` `enable`/`disable`/`commit`, track which client (surface) has an active text-input.
- When the focused surface has an active text-input, signal the overlay to show the OSK (via the S17-T6 event) and deliver `commit_string`/`preedit_string` from the overlay back to that `wlr_text_input_v3`.
- When text input is disabled, or the focused surface changes, signal the overlay to hide the OSK.
- Enforce the trust boundary: only the *focused* surface receives committed text; a background game never does.

**Done when:** a client calling `zwp_text_input_v3::enable` causes the compositor to emit the OSK-show signal, and `commit_string` reaches that client only while it is focused.

### S17-T3 — Raylib backend touch/pointer → `CORE.Input.Touch.*` / mouse

In `rcore_playos.c`:

- Add `wl_touch` and `wl_pointer` listeners on the seat (in addition to the existing keyboard listener in the shell's `src/input.c` — note the backend gets its own seat binding).
- Pointer: motion → `CORE.Input.Mouse.currentPosition`, button → `CORE.Input.Mouse.currentButtonState`, wheel → `CORE.Input.Mouse.currentWheelMove`. Populate `previousButtonState` on the next `PollInputEvents()`.
- Touch: down/up/motion → `CORE.Input.Touch.position[i]`, `CORE.Input.Touch.pointId[i]`, `CORE.Input.Touch.pointCount`, mapping `wl_touch` touch IDs to `MAX_TOUCH_POINTS` slots. Set `currentTouchState`/`previousTouchState` consistent with raylib's desktop backends.

**Done when:** `GetTouchPosition(0)`/`GetTouchPointCount()` return real values on the Ally touchscreen, and `GetMousePosition()` tracks a USB mouse.

### S17-T4 — Raylib backend text-input client + `ShowOnScreenKeyboard()`

- Bind `zwp_text_input_manager_v3` from the registry; create a `zwp_text_input_v3` and attach it to the seat.
- Add PlayOS raylib extensions:

```c
RLAPI void ShowOnScreenKeyboard(void);   // enable text input → compositor raises OSK
RLAPI void HideOnScreenKeyboard(void);   // disable text input → compositor hides OSK
```

- On `zwp_text_input_v3::commit_string`, push the UTF-8 string into raylib's `CORE.Input.Keyboard.charPressedQueue` (and optionally map a synthetic `Enter` keycode into `keyPressedQueue` for the "commit" key). `GetCharPressed()` then returns the typed characters exactly as if they came from a physical keyboard.
- Keep the existing "keyboard input is intentionally unhandled" stance for *physical* keyboards; only the text-input (OSK) path feeds the char queue.

**Done when:** a raylib game calling `ShowOnScreenKeyboard()` gets typed characters back through `GetCharPressed()`.

### S17-T5 — Overlay OSK UI + layout + tap-to-type

- Implement the OSK as a raylib UI component in `playos-overlay`. The draft cited `core_keyboard_testbed.c` as the starting point; that file does not exist. The real precedent to lift is the Sprint 16 passphrase keyboard in `playos-shell/src/screen_network.c` — key rectangles with centred glyphs, CAPS and a digits/symbols layer, a show/hide password toggle, and a layout sized from the panel dimensions so it fits rather than overflows.
- Layouts: a compact qwerty (rows: numbers, qwerty, asdf, zxcv + modifiers), plus numeric and password variants selected by content-hint from the compositor (see S17-T6).
- Modifiers: shift (capitalizes + swaps symbol layer), backspace, enter (commit), space, dismiss (hide OSK). Highlight the pressed key; repeat on hold is optional.
- Render above the game at the bottom of the panel; respect `output_info` dimensions/scale from the existing overlay protocol.

**Done when:** the OSK renders in the overlay, keys highlight on touch, and tapping emits the correct key/char commit request.

### S17-T6 — Overlay ↔ compositor OSK commit protocol

Extend `playos_overlay_v1` (in `playos-v1.xml`) with a minimal OSK channel. The overlay renders/hit-tests; the compositor mediates delivery:

```xml
<!-- Compositor → overlay -->
<event name="osk_visibility">
  <arg name="visible" type="uint" summary="1 = show OSK, 0 = hide"/>
  <arg name="hint"     type="uint" summary="content hint: normal|number|password|url"/>
</event>

<!-- Overlay → compositor -->
<request name="osk_commit_string">
  <arg name="text" type="string" summary="UTF-8 committed text"/>
</request>
<request name="osk_key">
  <arg name="key" type="uint" summary="special key: backspace|enter|tab|escape|dismiss"/>
</request>
```

- The compositor forwards `osk_commit_string` to the focused client's `wlr_text_input_v3` (`commit_string`) and `osk_key` to `keysym`/`done` events.
- Version the interface (bump `playos_overlay_v1` to `version="2"`; keep v1 clients working — additive).
- Regenerate headers with `wayland-scanner`; document in `wayland-protocol.md`.

**Done when:** tapping "A" in the OSK produces a `commit_string "A"` event on the focused client's text-input.

### S17-T7 — Shell text-field integration

- Add a shell text-field widget that, on focus, calls `ShowOnScreenKeyboard()`, renders the committed string, and calls `HideOnScreenKeyboard()` on submit/dismiss.
- Wire the Sprint 16 Wi-Fi passphrase field to it as the first real consumer (masked as password via the content hint).
- Confirm the shell uses the *same* OSK/text-input path as games (the shell is just another focused client).

**Done when:** selecting the Wi-Fi passphrase field raises the OSK, typed characters appear masked, and Enter commits the passphrase.

### S17-T8 — OSK sample game + end-to-end validation

- Add `com.playos.sample-osk`: a raylib game with a text field, a "show keyboard" button, and an echo area that renders `GetCharPressed()` output and touch coordinates.
- Validation matrix on the Ally:
  - Touch reaches the focused game (`GetTouchPosition` non-zero, correct quadrant).
  - Game invokes OSK via `ShowOnScreenKeyboard()`; keys type; text echoes in the game.
  - Shell invokes the same OSK for Wi-Fi passphrase; text is masked; Enter commits.
  - Background game receives **no** text while shell/overlay is focused.
  - Dismiss (B/system button) hides the OSK and returns focus to the game.
  - QEMU/CI: the compositor and raylib backend compile with `wlr_text_input_v3` and touch symbols. **Correction (2026-09-25, measured):** the emulator image's compositor receives **no input devices at all** — booting it with `virtio-keyboard-pci`, `virtio-mouse-pci` and `virtio-tablet-pci` produced neither a pointer nor even the pre-existing keyboard attach line. The minimal emulator image has no session for wlr's libinput backend, so QEMU cannot exercise this path as the image stands; an earlier note here claimed the pointer half was testable headlessly via `virtio-tablet`, and that was wrong. Either give the emulator image a session (seatd) or verify on hardware.

**Done when:** the sample echoes typed text on the Ally, and the shell Wi-Fi passphrase flow works end-to-end.

---

## Verification and Evidence

| Evidence | How it is produced |
|---|---|
| Touch reaches surface | Debug log of `wl_touch` down/up with surface-local coords |
| OSK raises on enable | `osk_visibility(1)` emitted when focused client enables text input |
| Text committed to focused client | `commit_string` received by the focused `wlr_text_input_v3` only |
| Game echo | `com.playos.sample-osk` echoes `GetCharPressed()` output |
| Shell passphrase | Wi-Fi passphrase field accepts masked text via OSK |
| Background isolation | Background game's `commit_string` count stays 0 |
| No reserved-key synthesis | OSK never emits `SYSTEM`/`QUICK_MENU` (static assert/audit) |
| CI build | wlroots + raylib compile with touch/text-input symbols; mock round-trip passes |

---

## Acceptance Criteria

- [ ] Touch taps on the ROG Ally reach the focused surface as raylib `GetTouchPosition()` points
- [ ] A USB mouse updates raylib `GetMousePosition()`/`IsMouseButton*()`
- [ ] The OSK can be invoked from a game (`ShowOnScreenKeyboard()`) and from the shell (Wi-Fi passphrase)
- [ ] Tapping OSK keys delivers `commit_string` to the *focused* client only
- [ ] Shift/caps, backspace, enter, space, and dismiss all behave correctly
- [ ] The shell Wi-Fi passphrase field is masked and submits via the OSK
- [ ] A background game receives no OSK text
- [ ] The OSK never synthesizes reserved `SYSTEM`/`QUICK_MENU` input
- [ ] `com.playos.sample-osk` echoes typed text and touch coordinates on hardware
- [ ] CI passes (touch/text-input symbols compile; mock text-input round-trip passes; headless touch path is inert)

---

## Handoff to Post-MVP

After this sprint, post-MVP features may assume:

- Touch and pointer are first-class seat inputs; games read them via standard raylib APIs
- A system OSK exists that any focused client (shell or game) can invoke via `zwp_text_input_v3`
- Text entry works for Wi-Fi passphrase, search, save naming, and user profiles
- Physical keyboard forwarding (a separate sprint) can build on the text-input focus routing added here

---

## Exit Gate

A finger tap lands in the focused surface, and the same system on-screen keyboard serves both the shell and games — invoked by the client, rendered by the overlay, and delivering committed text to the focused client through the standard `zwp_text_input_v3` protocol.

*Previous: [Sprint 16](Sprint-16.md) | Next: [Sprint 19](Sprint-19.md)*

---

## Realignment notes (2026-09-25)

Reviewed against the live tree before starting, the way Sprint 16 was.

| Claim in the draft | Reality | Action |
|---|---|---|
| "passphrase entry is currently blank/stubbed" | Sprint 16 shipped a working passphrase field **with** a hand-rolled keyboard (CAPS / 123-symbols / SHOW·HIDE), verified on the Ally | start conditions corrected; T7 becomes *replacing* that keyboard, not building text entry from nothing |
| touch "reaches the compositor" is assumed | **no touchscreen driver in `board/ally/linux.config`** — no `TOUCHSCREEN_*`, `I2C_HID` or `HID_MULTITOUCH` | T1 gained a kernel half; its primary repo and done-when updated |
| wlroots 0.20 provides `wlr_text_input_v3` / `wlr_cursor` | confirmed — both headers are in the pinned 0.20.0 tree | recorded in the start conditions |
| the compositor forwards no pointer/touch today | confirmed — no `wlr_cursor`, `wl_touch`, `wl_pointer` or `text_input` anywhere in `playos-compositor/src/` | T1 unchanged |
| the raylib backend needs touch/text-input work | confirmed — `rcore_playos.c` has no touch, pointer or text-input code | T3/T4 unchanged |
| OSK UI goes in `playos-overlay` under `src/` | the overlay is a single `main.c` + `CMakeLists.txt` + `protocols/`; **no `src/`**, and no `core_keyboard_testbed.c` | expected-files corrected; T5's starting point changed to the Sprint 16 keyboard |
| extend `playos_overlay_v1` in `playos-runtime/protocols/playos-v1.xml` | confirmed — the file exists and declares `playos_overlay_v1 version="1"` | T6 unchanged |
| adding a sample under `playos-samples` is enough | the package builds from an **explicit list** in `playos-samples.mk`; no glob | `playos-refdistro` added to required changes and expected files |
| "no touch device present → path is inert" in CI | virtio-input is already enabled in the emulator kernel, so the **pointer** half is testable via `virtio-tablet`/`virtio-mouse` | CI line corrected so the sprint does not under-test |

The sprint's shape is unchanged: three layers (compositor → raylib backend →
overlay OSK) plus two consumers. What changed is T1's starting point — **until the
kernel can see the panel, no amount of compositor work is verifiable on hardware.**

---

## Re-scope (2026-09-26) — after the confirmed product direction

Every game and app is SDK-built, so there are no foreign Wayland clients and path 2 is not
planned. Where this differs from the task framing above, this is the work that is actually
required:

1. **Touch in the platform API** (`playos-platform-api/src/backends/backend_evdev.c`): read the
   panel's `ABS_MT_*` events and expose them the way the gamepad already is. The kernel half is
   done and verified on hardware (`i2c_hid_acpi` + `hid-multitouch`, `/dev/input/event5`).
2. **Keyboard and mouse in the platform API** — PlayOS targets PC hardware, where browsing and
   searching a marketplace makes them first-class rather than optional.
3. **Text entry as a PlayOS API**: a client requests text, an OSK is raised, committed text
   returns. The OSK is **overlay-rendered** (the shell stops drawing while a game is foreground)
   and built from LVGL's keyboard/textarea widgets (Sprint 22 / ADR-0013) rather than hand-drawn.
4. **Consumers**: the shell's and apps' own fields, plus the sample that echoes typed text (T8
   stands as the acceptance test).

Retained but inert: the seat forwarding in `playos-compositor/src/input.c` (T1's second half,
which is correct and needs no further work now) and `zwp_text_input_v3` (T2/T4) — the
implementation for a future client class that does not exist today.

**Verified end to end (2026-09-26):** touch reaches an SDK application on hardware. The platform
API found the panel (`NVTK0603:00 0603:F200`, multitouch) and a probe recorded 17 finger-downs
including 7 two-finger contacts; then a raylib game launched normally from the library (sandboxed
as `playos-game`) drew a circle under every finger and reported up to **7-8 simultaneous points**
- raylib's `MAX_TOUCH_POINTS` ceiling, reached on real hardware. The path is `GetTouchPosition()`
-> `libplayos` -> evdev, with no compositor and no Wayland involvement (ADR-0013).

