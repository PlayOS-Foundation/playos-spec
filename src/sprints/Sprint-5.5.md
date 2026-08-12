# Sprint 5.5 — Shell → Raylib 6.0 Migration

**Goal:** Resolve the divergence between Sprint 5's *specified* rendering framework (Raylib, per ADR-0006 and `playos-shell-spec.md`) and the *actual* shell implementation (raw EGL/GLES2 with a hand-rolled shader + bitmap font). Upgrade the vendored Raylib from 5.5 to **6.0**, implement the custom `rcore_playos.c` platform backend against the 6.0 backend contract, and port the shell's rendering to the Raylib draw API — without changing the shell's UX, input model, or lifecycle behaviour.

**Primary Outcome:** The ROG Ally boots into the same four-screen, controller-first shell, but every frame is now drawn through Raylib 6.0 via a custom Wayland/EGL platform backend. The raw-GLES2 renderer in `render_util.c` is retired. `PLAYOS_SHELL_USE_RAYLIB=ON` is the Buildroot default, and Raylib 6.0 is pinned in `versions.lock`.

**Status:** 🟡 Planned — migration pending

**Prerequisites:** Sprint 5 complete — the shell renders and navigates on the Ally through the raw EGL/GLES2 path (4 screens, controller-first input, frame-callback vsync, lifecycle handling, per-call trusted IPC). Raylib 5.5 is vendored at `playos-shell/external/raylib/` but unused. The custom backend contract is specified in ADR-0006.

---

## Why This Sprint Exists

Sprint 5 ended in a spec/implementation divergence that, left unaddressed, will compound through Sprint 6 (storage/game discovery), Sprint 7 (launch/lifecycle), and Sprint 8 (overlay):

1. **Spec says Raylib; code says raw GLES2.** `playos-shell-spec.md` and ADR-0006 commit to a Raylib shell with a custom `rcore_playos.c` backend. The implementation deferred Raylib and instead wrote a bespoke single-shader GLES2 renderer plus a hand-embedded 5×7 bitmap font in `render_util.c`.
2. **The vendored Raylib is stale and inert.** `playos-shell/external/raylib/` vendors **Raylib 5.5**, but `PLAYOS_SHELL_USE_RAYLIB` is `OFF` in both `CMakeLists.txt` and the Buildroot package (`playos-shell.mk`). The `versions.lock` entries `RAYLIB_COMMIT` / `RAYLIB_SOURCE` are empty.
3. **Raylib 6.0 changes the platform backend contract.** The backend split introduced in Raylib 5.0 was reworked in 6.0, along with module-level compile toggles and symbol renames/removals. The custom PlayOS backend must be written against 6.0, not 5.5.
4. **A hand-rolled renderer becomes a maintenance trap.** The raw-GLES2 path (custom shaders, bitmap font, manual text metrics) must be re-implemented in Raylib eventually anyway. Migrating now, before Sprint 6/7 build on the rendering path, avoids double work and keeps the shell aligned with ADR-0006's "Raylib for shell, overlay, and games" decision.

This is a **pure migration sprint** — no new screens, no new features, no UX changes. The user-visible result must be identical (or strictly better) to Sprint 5.

---

## Start Condition Checklist

- [ ] Sprint 5 shell renders and navigates on the Ally via raw EGL/GLES2. *(Verified at Sprint 5 exit gate.)*
- [ ] Raylib 5.5 is vendored at `playos-shell/external/raylib/` and builds (or the `PLAYOS_SHELL_USE_RAYLIB` path is understood).
- [ ] `versions.lock` has empty `RAYLIB_COMMIT` / `RAYLIB_SOURCE` entries ready to be filled.
- [ ] Raylib 6.0 release source/tag is available to vendor (exact commit SHA obtainable).
- [ ] Nested Wayland dev environment and QEMU headless path are usable for iteration.

---

## Decisions Locked for This Sprint

- **Raylib version:** **6.0**, vendored into `playos-shell/external/raylib/` and pinned with a full commit SHA in `versions.lock` under `RAYLIB_COMMIT`.
- **Backend:** a custom `rcore_playos.c` platform backend (not GLFW/SDL backends). It owns the fullscreen `xdg_toplevel`, `wl_egl_window`, EGL/GLES2 context, and frame-callback vsync — the same primitives `main.c` currently manages directly.
- **Raylib is rendering-only.** Input remains shell-owned direct evdev (`input.c`) so the reserved SYSTEM/QUICK_MENU buttons are preserved. Raylib's gamepad abstraction is **not** used for navigation because it strips reserved buttons.
- **Module stripping:** disable unused Raylib subsystems via `config.h` `SUPPORT_MODULE_*` toggles (audio `raudio`, models `rmodels`, camera `rcamera`, and networking if present), keeping `rcore`, `rlgl`, `rshapes`, `rtext`, and `rtextures`. This avoids conflicts with the Sprint 8 ALSA stack and keeps the shell binary small.
- **Font:** adopt Raylib's default font (`GetFontDefault()`); remove the embedded 5×7 bitmap font from `render_util.c`.
- **No new features.** UX, screens, navigation, and lifecycle behaviour are unchanged from Sprint 5.

---

## Scope

### In Scope

- Upgrade vendored Raylib 5.5 → 6.0 and pin it in `versions.lock`
- Reconcile Raylib 6.0 breaking changes (platform backend contract, module flags, renamed/removed symbols)
- Implement `rcore_playos.c` against the Raylib 6.0 backend contract
- Port rendering from raw GLES2 (`render_util.c`) to the Raylib draw API across all four screens
- Wire shell-owned evdev input and lifecycle polling into the Raylib frame loop
- Buildroot packaging: flip `PLAYOS_SHELL_USE_RAYLIB=ON`, add the vendored Raylib dependency
- Spec/doc reconciliation (`playos-shell-spec.md`, `playos-shell/AGENTS.md`, ADR-0006 note)
- Validation in the nested Wayland dev path, QEMU headless path, and on the Ally

### Explicitly Out of Scope

- New screens, navigation flows, or UX changes
- Game launch/resume/background lifecycle (Sprint 7)
- Overlay UI (Sprint 8)
- Audio via Raylib `raudio` (Sprint 8 uses ALSA)
- 3D/model rendering (`rmodels` is stripped)
- Intel graphics expansion (Sprint 13)
- Store/network/account features

---

## Required Repository Changes

| Repo | Required work |
|---|---|
| `playos-shell` | Vendor Raylib 6.0, implement `rcore_playos.c`, port `render_util.c` + `screen_*_draw()` to the Raylib API, wire input/lifecycle, flip `PLAYOS_SHELL_USE_RAYLIB`, update `AGENTS.md` |
| `playos-refdistro` | Pin `RAYLIB_COMMIT` in `versions.lock`, flip `playos-shell.mk` to `USE_RAYLIB=ON`, ensure the vendored Raylib builds/links in the Buildroot image |
| `playos-spec` | Add `Sprint-5.5.md`, update `playos-shell-spec.md`, add ADR-0006 follow-up note, update cross-links |

---

## Expected Files and Directories

### `playos-shell` (changed/added)

```text
external/raylib/
├── src/raylib.h             ← UPDATE: RAYLIB_VERSION "6.0"
└── src/platforms/
    └── rcore_playos.c       ← NEW: custom PlayOS Wayland/EGL backend (6.0 contract)

src/
├── main.c                   ← UPDATE: delegate surface/EGL to the raylib backend
├── input.c                  ← UNCHANGED: shell-owned direct evdev (trusted reserved buttons)
├── render_util.c            ← REWRITE: thin Raylib wrappers (no raw GLES2 shader/font)
├── screen_home.c            ← UPDATE: draw via Raylib API
├── screen_library.c         ← UPDATE: draw via Raylib API
├── screen_game_detail.c     ← UPDATE: draw via Raylib API
└── screen_settings.c        ← UPDATE: draw via Raylib API

CMakeLists.txt               ← UPDATE: build Raylib 6.0 with minimal SUPPORT_MODULE_* config
```

### `playos-refdistro`

```text
versions.lock                ← UPDATE: RAYLIB_COMMIT=<sha>, RAYLIB_SOURCE confirmed
br2-external/package/playos-shell/
└── playos-shell.mk          ← UPDATE: -DPLAYOS_SHELL_USE_RAYLIB=ON
```

### `playos-spec`

```text
src/playos-shell-spec.md     ← UPDATE: reflect Raylib 6.0 + rcore_playos.c
src/adr/ADR-0006-*.md        ← ADD: follow-up note (or new superseding ADR) if the decision changed
src/sprints/Sprint-5.5.md    ← NEW: this document
```

---

## Agent Task Breakdown

Every task below is independently checkable.

### Task Status Grid

| Task ID | Task | Primary repo | Status | Notes / evidence |
|---|---|---|---|---|
| S5.5-T1 | Upgrade vendored Raylib 5.5 → 6.0 and pin it | `playos-shell`, `playos-refdistro` | not started | `external/raylib/` currently 5.5; `versions.lock` `RAYLIB_COMMIT` empty |
| S5.5-T2 | Reconcile Raylib 6.0 breaking changes | `playos-shell` | not started | Produce and apply a 5.5 → 6.0 migration list |
| S5.5-T3 | Implement `rcore_playos.c` platform backend (6.0) | `playos-shell` | not started | Custom Wayland/EGL backend per ADR-0006 |
| S5.5-T4 | Port rendering from raw GLES2 to Raylib draw API | `playos-shell` | not started | Retire shader + bitmap font in `render_util.c` |
| S5.5-T5 | Wire controller input (rendering-only Raylib) | `playos-shell` | not started | Keep `input.c` evdev; no Raylib gamepad |
| S5.5-T6 | Integrate lifecycle polling into the Raylib frame loop | `playos-shell` | not started | Suspend skips draw; TERMINATE exits cleanly |
| S5.5-T7 | Buildroot packaging: flip to Raylib 6.0 | `playos-refdistro` | not started | `USE_RAYLIB=ON`; vendored raylib links in image |
| S5.5-T8 | Spec and docs reconciliation | `playos-spec`, `playos-shell` | not started | Remove "Raylib deferred" language everywhere |
| S5.5-T9 | Validation and runtime evidence | `playos-shell`, `playos-refdistro` | not started | dev / QEMU / Ally paths verified |

---

### S5.5-T1 — Upgrade Vendored Raylib 5.5 → 6.0 and Pin It

**Finding:** `playos-shell/external/raylib/` vendors Raylib 5.5 (`RAYLIB_VERSION "5.5"` in `src/raylib.h`). `versions.lock` has `RAYLIB_COMMIT=` and `RAYLIB_SOURCE=https://github.com/raysan5/raylib` but no pinned commit. `PLAYOS_SHELL_USE_RAYLIB` is `OFF` everywhere.

**Steps:**

1. **Replace the vendored source.** Swap `external/raylib/` with the Raylib **6.0** release (exact tag or commit), keeping the repo's vendoring layout intact.
2. **Pin it.** Set `RAYLIB_COMMIT=<full sha>` in `playos-refdistro/versions.lock` (confirm `RAYLIB_SOURCE` is correct). Do not use a branch or `latest`.
3. **Update the CMake integration.** In `playos-shell/CMakeLists.txt`, adjust the `add_subdirectory(external/raylib)` path and the library target name if Raylib 6.0 renamed its CMake target (e.g., `raylib` vs `raylib_playos`). Keep `PLAYOS_SHELL_USE_RAYLIB` as the gating option.
4. **Apply a minimal module config.** Configure `config.h` `SUPPORT_MODULE_*` toggles to strip unused subsystems (see Decisions). This keeps the vendored build lean and avoids pulling in `raudio`/`rmodels` deps.
5. **Standalone build check.** Build the vendored Raylib alone (`cmake -B build && cmake --build build`) to confirm it compiles with the stripped config before touching the shell.

**Done when:**
- `external/raylib/src/raylib.h` reports `RAYLIB_VERSION "6.0"`.
- `versions.lock` has a non-empty `RAYLIB_COMMIT`.
- The vendored Raylib builds standalone with the minimal module config.

---

### S5.5-T2 — Reconcile Raylib 6.0 Breaking Changes

**Finding:** Raylib 6.0 reworks the platform-backend contract introduced in 5.0, adds module-level compile toggles, and renames/removes a number of symbols. The shell cannot adopt 6.0 blindly — it must know exactly what changed between 5.5 and 6.0.

**Steps:**

1. **Diff 5.5 → 6.0.** Read the upstream `CHANGELOG` (and the platform-backend headers) and produce a concrete reconciliation list covering:
   - platform backend entry points / `PLATFORM_*` contract changes
   - `config.h` flag renames and new `SUPPORT_MODULE_*` toggles
   - renamed/removed public symbols (window, input, drawing, text, math)
   - GLES2 / `rlgl` renderer API changes relevant to the custom backend
2. **Apply the list.** Update any shell references (current or planned) that use a renamed/removed 5.5 symbol. Since Raylib is not yet wired into the shell, this mostly informs T3/T4, but any `external/` integration glue is corrected here.
3. **Record it.** Capture the final list in this sprint's evidence (or an ADR-0006 follow-up note) so Sprint 6/7 do not re-discover the same changes.

**Done when:**
- A reviewed 5.5 → 6.0 breaking-change list exists.
- No shell or integration code references a removed 5.5-only symbol.

---

### S5.5-T3 — Implement the `rcore_playos.c` Platform Backend (Raylib 6.0)

**Finding:** ADR-0006 mandates a custom Raylib backend that integrates with the PlayOS Wayland/EGL environment and the `libplayos` lifecycle API. It must now be written against the **6.0** backend contract in `src/platforms/rcore_playos.c`.

**Steps:**

1. **Implement `rcore_playos.c`** against Raylib 6.0's platform backend contract:
   - create a fullscreen `xdg_toplevel` (reuse the shell's `xdg_wm_base`/`wl_compositor` setup)
   - create a `wl_egl_window` + EGL/GLES2 context and make it current
   - pace frames with Wayland frame callbacks (`wl_surface_frame`) and present with `eglSwapBuffers`
2. **Disable desktop features** as no-ops: window decorations, free resize, drag-and-drop, clipboard, multi-window.
3. **Wire the trusted-shell registration.** Keep the `playos_manager_v1` "register as trusted shell" + ShellReady handshake from `main.c`, now invoked through/alongside the backend init.
4. **Expose dimensions.** Ensure `GetScreenWidth()` / `GetScreenHeight()` reflect the toplevel configure size (the shell's `dpi_scale` convention can map onto Raylib scaling).
5. **Trim `main.c`.** Remove the now-duplicated direct EGL/`wl_surface`/frame-callback management that the backend owns.

**Done when:**
- A Raylib frame draws through `rcore_playos.c` in the nested Wayland dev environment.
- `main.c` no longer manages the EGL surface/context directly.

**On-device fix (frame pacing):** The first Ally bring-up froze after `entering main loop` — the shell logged no FPS lines and the screen showed only the compositor's background. Root cause: `SwapScreenBuffer()` armed a `wl_surface_frame` callback and committed the surface *before* `eglSwapBuffers` attached a buffer. On frame 1 the surface is still unmapped (no buffer), so the compositor never presents it and the frame callback never fires — `wl_display_dispatch` blocks forever. Fix (`playos-shell@02fa9fa`): reorder to the canonical wait-after-swap pattern — wait for the previous frame's callback, then `eglSwapBuffers` (which attaches the buffer and commits), then arm the callback for the frame just presented.

---

### S5.5-T4 — Port Rendering from Raw GLES2 to the Raylib Draw API

**Finding:** `render_util.c` implements a single-shader GLES2 renderer plus an embedded 5×7 bitmap font. The four `screen_*_draw()` functions call `render_draw_rect()` / `render_draw_text()`. These must map onto Raylib.

**Steps:**

1. **Map each `render_*` helper to Raylib:**

| Current helper | Raylib equivalent |
|---|---|
| `render_init(w, h)` | backend window/context init (T3) |
| `render_begin_frame(r,g,b,a)` | `BeginDrawing()` + `ClearBackground()` |
| `render_draw_rect(x,y,w,h,r,g,b,a)` | `DrawRectangle()` / `DrawRectangleRec()` |
| `render_draw_text(text,x,y,scale,r,g,b,a)` | `DrawTextEx()` |
| `render_end_frame(s)` | `EndDrawing()` |
| `render_screen_dims(&w,&h)` | `GetScreenWidth()` / `GetScreenHeight()` |
| `render_text_width(text, scale)` | `MeasureTextEx()` |

2. **Replace the font.** Use `GetFontDefault()`; delete the embedded `font_data` array and GLSL shader sources from `render_util.c`.
3. **Update the four `screen_*_draw()` functions.** Either keep the thin `render_*` wrappers (now backed by Raylib) or call Raylib directly. Preserve each screen's layout and focus-visibility rules.
4. **Remove dead code.** Delete the now-unused GLES2 quad/shader/text-metric code.

**Done when:**
- All four screens render through Raylib.
- `render_util.c` contains no raw GLES2 shader or bitmap-font code.

---

### S5.5-T5 — Wire Controller Input (Rendering-Only Raylib)

**Finding:** The shell reads controller input directly from evdev in `input.c` because the reserved SYSTEM/QUICK_MENU buttons must survive (they are stripped by the `libplayos` input API). Raylib must be used for rendering only.

**Steps:**

1. **Keep `shell_input_poll()` and edge detection** in `input.c` as the single source of controller state — do not route navigation through Raylib's gamepad abstraction.
2. **Confirm the invariants** after the rendering migration: `A` confirms, `B` backs out, d-pad moves focus, focus is always visible.
3. **Verify** SYSTEM/QUICK_MENU still arrive (these buttons do not exist in Raylib's gamepad mapping and must stay on the shell's evdev path).

**Done when:**
- Controller navigation on all four screens is unchanged while rendering through Raylib.
- Reserved buttons remain available to the trusted shell.

---

### S5.5-T6 — Integrate Lifecycle Polling into the Raylib Frame Loop

**Finding:** `main.c` polls `playos_lifecycle_poll()` every frame and handles SUSPEND/RESUME/FOREGROUND/BACKGROUND/TERMINATE. This must survive the Raylib migration.

**Steps:**

1. **Keep lifecycle polling** in the Raylib main loop (or in the backend's per-frame hook).
2. **Suspend/background** → skip `BeginDrawing()`/`EndDrawing()` (no rendering work). **Foreground/resume** → resume normal rendering.
3. **TERMINATE** → exit cleanly through the Raylib window-close path (`s->running = false`).
4. **Confirm** the shell remains a persistent supervised process (no `exit()` except on unrecoverable init failure).

**Done when:**
- Lifecycle state changes affect rendering predictably.
- The shell exits cleanly on TERMINATE and remains compatible with `playos-init` supervision.

---

### S5.5-T7 — Buildroot Packaging: Flip to Raylib 6.0

**Finding:** `playos-shell.mk` passes `-DPLAYOS_SHELL_USE_RAYLIB=OFF`. The vendored Raylib is inside the `playos-shell` source tree but not built into the image.

**Steps:**

1. **Flip the flag.** Set `-DPLAYOS_SHELL_USE_RAYLIB=ON` in `br2-external/package/playos-shell/playos-shell.mk`.
2. **Ensure the vendored Raylib builds and links.** Confirm the `playos-shell` CMake builds the vendored `external/raylib` static library and links it into `playos-shell`. Add any missing dependency to `PLAYOS_SHELL_DEPENDENCIES` (Raylib stays vendored inside `playos-shell`, so it should not need a separate Buildroot package).
3. **Keep libplayos + GLES/EGL/Wayland** through the backend. Update the `.mk` header comment (it currently says "Uses EGL/GLES2 for rendering" — change to "Raylib 6.0 via custom PlayOS backend").
4. **Pin in `versions.lock`** (done in T1) and confirm `make setup` vendored the pinned commit.
5. **Build.** Run `make qemu-build` and confirm the image contains the Raylib-linked shell.

**Done when:**
- `playos-shell.mk` sets `USE_RAYLIB=ON`.
- The QEMU image builds and boots with the Raylib-backed shell.

---

### S5.5-T8 — Spec and Docs Reconciliation

**Finding:** `playos-shell-spec.md` says "Rendering with Raylib PlayOS backend", but `playos-shell/AGENTS.md` says "Raylib integration deferred; direct EGL/GLES2 used instead". ADR-0006's consequences require the `rcore_playos.c` backend to be maintained across Raylib updates.

**Steps:**

1. **`playos-shell-spec.md`:** update the rendering responsibilities and cross-references to name Raylib 6.0 + `rcore_playos.c`; remove any "deferred" wording.
2. **`playos-shell/AGENTS.md`:** update the implementation-status banner and the "Rendering" section to state Raylib 6.0 is active via the custom backend (raw GLES2 path retired).
3. **ADR-0006:** add a follow-up note (or a superseding ADR if the decision materially changed — e.g., the explicit "rendering-only Raylib, input stays shell-owned evdev" ruling) recording the 6.0 migration.
4. **Cross-links:** this document's footer already links Previous/Next; add any in-doc references from Sprint 5/6 where useful (mirroring how Sprint 2.5 is referenced by Sprint 2).

**Done when:**
- No documentation claims "Raylib deferred" or describes the raw-GLES2 renderer as the active path.

---

### S5.5-T9 — Validation and Runtime Evidence

**Steps:**

1. **Nested Wayland dev run:** shell boots, draws all four screens through Raylib, navigates with controller, logs manifest load + navigation + fps.
2. **QEMU headless path:** shell starts without crashing (visual output limited, but must not regress from Sprint 5).
3. **On-device (Ally):** visual parity with Sprint 5, ≥ 60 fps, controller navigation, persistence under supervision.
4. **Capture evidence:** `/data/log/shell.log` showing Raylib version, backend init, GPU string, and fps.

**Done when:**
- Evidence is captured across dev / QEMU / device paths.
- All acceptance criteria below are verified.

---

## Implementation Guidance

### Order of execution

1. **T1 first** (vendor + pin Raylib 6.0) — everything depends on the new source.
2. **T2 second** (reconciliation list) — informs the backend and port work.
3. **T3 third** (backend) — the foundation the port sits on.
4. **T4 fourth** (rendering port) — the largest surface change, across all screens.
5. **T5, T6 next** (input, lifecycle) — wire the shell's existing behaviours into the Raylib loop.
6. **T7 sixth** (Buildroot) — build the whole image.
7. **T8 seventh** (docs) — document reality after the code lands.
8. **T9 last** (validation) — verify across all three runtime paths.

### Atomic commits

Each task is a separate commit (or small group) referencing the task ID:

```
S5.5-T1: vendor raylib 6.0 and pin in versions.lock
S5.5-T3: implement rcore_playos.c backend for raylib 6.0
S5.5-T4: port shell rendering to raylib draw API
```

### Do not break the shell

After T1–T7, the shell must still render all four screens, navigate with a controller, and stay alive under supervision. If the raw-GLES2 path was already broken before this sprint, document it and fix only what this sprint touches.

---

## Verification and Evidence

| Evidence | How it is produced |
|---|---|
| Raylib 6.0 proof | `external/raylib/src/raylib.h` shows `RAYLIB_VERSION "6.0"`; `versions.lock` `RAYLIB_COMMIT` non-empty |
| Backend proof | a Raylib frame draws through `rcore_playos.c` in the nested Wayland dev path |
| Rendering port proof | `render_util.c` contains no raw GLES2 shader/bitmap-font code; all four screens use the Raylib API |
| Input proof | controller navigation unchanged; reserved SYSTEM/QUICK_MENU buttons still delivered to the shell |
| Lifecycle proof | suspend skips rendering, foreground resumes, TERMINATE exits cleanly |
| Packaging proof | `playos-shell.mk` sets `USE_RAYLIB=ON`; QEMU image boots the Raylib shell |
| Docs proof | `playos-shell-spec.md`, `AGENTS.md`, ADR-0006 no longer describe Raylib as deferred |
| Runtime proof | shell log shows Raylib version + backend init + ≥ 60 fps on the Ally |

---

## Acceptance Criteria

- [ ] vendored Raylib reports `RAYLIB_VERSION "6.0"` and is pinned in `versions.lock`
- [ ] Raylib builds with the minimal `SUPPORT_MODULE_*` module config
- [ ] a Raylib 5.5 → 6.0 breaking-change reconciliation list exists and is applied
- [ ] `rcore_playos.c` implements the Raylib 6.0 platform backend (fullscreen Wayland + EGL + frame callback)
- [ ] all four screens render through the Raylib draw API (no raw GLES2 shader/bitmap font in `render_util.c`)
- [ ] controller navigation is unchanged and uses shell-owned evdev (not Raylib's gamepad abstraction)
- [ ] lifecycle suspend skips rendering; foreground resumes; TERMINATE exits cleanly
- [ ] `PLAYOS_SHELL_USE_RAYLIB=ON` in Buildroot; the image builds and boots the Raylib shell
- [ ] `playos-shell-spec.md`, `playos-shell/AGENTS.md`, and ADR-0006 reflect Raylib 6.0 (no "deferred")
- [ ] ≥ 60 fps on the Ally; nested dev and QEMU paths remain usable for iteration

---

## Handoff to Sprint 6

Sprint 6 may assume:

- The shell renders through Raylib 6.0 via `rcore_playos.c` — no raw-GLES2 fallback remains.
- The custom backend owns Wayland surface + EGL context + frame pacing; `main.c` delegates to it.
- Input remains shell-owned direct evdev (reserved buttons preserved); Raylib is rendering-only.
- Raylib is pinned in `versions.lock` and builds with a minimal module config.
- All spec/docs describe the active Raylib path, so Sprint 6 (storage/game discovery) builds on Raylib rendering rather than a hand-rolled renderer.

Sprint 6 should not need to touch the rendering layer except to display discovered game metadata through the existing Raylib draw path.

---

*Previous: [Sprint 5](Sprint-5.md) | Next: [Sprint 6](Sprint-6.md)*
