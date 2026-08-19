# Sprint 22 — LVGL Shell UI Spike (Post-MVP)

**Goal:** Prove that [LVGL](https://lvgl.io/) v9 can render a resolution-adaptive, controller-navigated shell UI *inside* the existing PlayOS shell, starting with a raylib-managed texture as a **smoke-test backend** — without replacing `rcore_playos.c`, the Wayland/EGL lifecycle, or the game ABI. This is a **bounded spike**, not a product-direction port. The final rendering path (CPU blit vs. GPU draw unit vs. `LV_USE_WAYLAND` full port) is decided at implementation time.

**Primary Outcome:** A working experimental LVGL screen navigated with the controller in the dev environment, plus a written **go/no-go** recommendation for a full LVGL shell port — including an explicit rendering-path decision (Path 1 / 2 / 3).

**Status:** 🟡 Post-MVP — spike defined; not started. No implementation work is approved until this sprint is scheduled.

**Prerequisites:** MVP stable (Sprint 15–16); the Raylib 6.0 shell landed (Sprint 5.5); `rcore_playos.c` is the single shell rendering backend (ADR-0006); the musl-only constraint is in force (ADR-0003); the nested-Wayland dev environment works (`playos-spec/src/dev-environment.md`).

---

## Why This Sprint Exists

The shell is currently a controller-first raylib application that draws rectangles and text with immediate-mode calls (`render_util.c`). Building a polished, screen-adaptive UI this way is possible but laborious: layout is hand-derived from `GetScreenWidth()/GetScreenHeight()`, and every widget (list, grid, focus state, transition) is bespoke.

LVGL is a retained-mode embedded UI library with widgets, themes, animations, flexbox/grid layout, and DPI scaling — a materially better toolkit for a *screen-based, resolution-independent, controller-navigated* shell. The open question is not "is LVGL capable" but "can it be integrated into the existing raylib-backed shell at low risk."

This sprint answers that with a **smoke test first, then a deliberate rendering-path decision**. The cheapest validation is a CPU texture-blit integration: LVGL renders to its own framebuffer, and its `flush_cb` uploads pixels into a raylib `Texture2D`; raylib draws that texture as a fullscreen quad. This keeps the entire Wayland/EGL/vsync/input stack intact and turns raylib into a thin blitter + event loop — enough to prove the UI value and input mapping, but not the production renderer.

---

## Assessment Inputs

The spike rests on the following authoritative facts:

- **LVGL v9.5.x, MIT, C99.** Royalty-free and static-link friendly; no OS dependency beyond a display buffer and a tick source. ([license](https://docs.lvgl.io/master/introduction/license.html), [requirements](https://docs.lvgl.io/master/introduction/requirements.html))
- **LVGL resolution model.** `lv_dpx()` DPI scaling, percentage units, `LV_SIZE_CONTENT`, min/max sizing, and flexbox/grid layouts provide responsive UI without hardcoded pixels. ([coordinates](https://docs.lvgl.io/master/common-widget-features/coordinates.html))
- **LVGL display driver hook.** A display is registered with a `flush_cb` that receives dirty pixel areas; the callback owns how pixels reach the framebuffer/GPU. This is the exact seam for a raylib texture backend.
- **LVGL input model.** Four input-device types exist — `POINTER`, `KEYPAD`, `ENCODER`, `BUTTON` — with **no native gamepad type**. Controller navigation is achieved by mapping d-pad/face buttons to `LV_KEY_*` and using `lv_group`/`lv_gridnav` focus. ([indev](https://docs.lvgl.io/master/main-modules/indev/overview.html), [groups](https://docs.lvgl.io/master/main-modules/indev/groups.html))
- **Current shell stack.** `external/raylib/src/platforms/rcore_playos.c` owns Wayland/EGL/GLES2 and frame-callback vsync; `src/input.c` reads controller evdev directly; `src/render_util.c` wraps raylib draw calls; the shell must stay alive at 60 fps with controller-only navigation and no blocking I/O on the render thread (`playos-shell/AGENTS.md`).
- **Raylib texture upload.** A `Texture2D` can receive CPU pixels via `UpdateTexture`, and sub-rectangle updates can use the GL texture id directly for `glTexSubImage2D`.

---

## Assessment Constraints (Locked)

These constraints are not re-negotiated by this sprint:

- **Single rendering backend for the shell stays raylib** ([ADR-0006](../adr/ADR-0006-ui-framework.md)) **unless Path 3 is chosen, in which case an ADR decision is required first.** In the default Path 1/2 spike, LVGL is a *UI layer on top of* raylib, not a second Wayland/EGL backend.
- **musl-only** ([ADR-0003](../adr/ADR-0003-libc-choice.md)) and **controller-only navigation** remain in force.
- **Shell invariants**: always alive, 60 fps target, no blocking I/O on the render thread, no direct IPC socket access except the trusted evdev input path, rendering stops while a game is foreground but the process stays alive.
- **No change to `rcore_playos.c`, the game ABI, or ADR-0006.** The spike is additive and reversible.

---

## Feasibility Assessment

### Verdict

**Feasible; low-to-moderate effort; low architectural risk.** The spike starts with the cheapest correctness check — a CPU-buffer texture blit — and treats that only as a **smoke test**, not the production renderer. The final rendering path is deliberately left open: at implementation time we may stay on the smoke-test path, or jump straight to one of the two GPU-accelerated paths below (`lv_opengles_texture` + GPU draw unit, or the `LV_USE_WAYLAND` full port).

### Rendering-path decision (re-evaluate at implementation time)

Three paths are on the table. The sprint does **not** lock us to the CPU blit.

| Path | How it works | Risk | Role in this sprint |
|---|---|---|---|
| **1 — CPU blit (smoke test)** | LVGL software renderer writes a CPU framebuffer; `flush_cb` uploads dirty pixels into a raylib `Texture2D`; raylib draws the fullscreen quad | Lowest | **Minimum first check only** — proves LVGL, input mapping, and the dev loop; not the target renderer |
| **2 — `lv_opengles_texture` + GPU draw unit** | LVGL rasterizes through a GPU draw unit (NanoVG preferred, or the GLES texture-cache unit) and hands back a GL texture that raylib blits; the shell keeps its EGL context and frame pacing | Low-to-medium | **Likely production path** — GPU-accelerated while retaining `rcore_playos.c` |
| **3 — `LV_USE_WAYLAND` full port** | LVGL owns the Wayland surface/EGL/vsync; drop `rcore_playos.c` for the shell | Medium | **Long-term option** — requires an ADR decision because it supersedes part of ADR-0006 |

Rule for implementation: run Path 1 as the cheapest first validation. If the smoke test is clean, do **not** assume Path 1 is the end state — explicitly re-evaluate and choose Path 2 (or Path 3 with an ADR) before any production work. We may go straight to Path 2 or 3 from the outset if that is more expedient.

Path 1 keeps raylib in control of GL state, vsync, and input, so LVGL's software renderer cannot fight raylib's GL state machine — which is exactly why it is a safe smoke test. Path 2 keeps that same shell ownership while adding GPU rasterization through LVGL's draw unit; it is the least disruptive GPU path and the likely production choice. Path 3 is the cleanest long-term architecture but re-owns Wayland/EGL, so it is a deliberate ADR decision, not a spike default.

### Plus / Minus

| Dimension | Plus | Minus |
|---|---|---|
| Resolution adaptation | `lv_dpx`, `%`, flex/grid, min/max — real responsive layouts | Breakpoints and assets still need design/testing |
| Look & feel | Widgets, themes, styles, animations; retained-mode reduces drawing code | Default themes are plain; a console-grade look needs custom theme/fonts/images |
| Integration risk | No `rcore_playos.c` change for Path 1/2; additive, reversible | Path 1 has two rendering stacks (raylib blit + LVGL software render); Path 3 replaces the shell backend and needs an ADR |
| Input | `lv_group`/`lv_gridnav` maps cleanly to D-pad | No native gamepad indev; custom evdev→`LV_KEY_*` glue required |
| Build | Pure C99, musl-safe | No Buildroot package; spike vendors LVGL under `playos-shell/external/` and defers Buildroot packaging |
| Footprint/performance | Tiny; partial refresh; dirty-area upload; Path 2 offloads rasterization to GPU | Path 1 is CPU-bound: 1080p@60 full-frame upload ~500 MB/s if not using dirty areas; must use sub-rect updates |
| Total | Faster path to a polished shell UI than hand-rolled raylib | One-time integration detail (format/endianness/upload/tick) plus a learning curve |

### Smoke-test integration design (Path 1 — CPU blit)

> This is the **minimum first check only**. It is not the production renderer. If Path 2 or 3 is chosen at implementation time, the `flush_cb` below is replaced by the GPU draw-unit or `LV_USE_WAYLAND` driver respectively.

```c
/* One full-screen 32-bit RGBA8 draw buffer (v9 API names to be checked). */
static lv_color_t fb[SCREEN_W * SCREEN_H];
static Texture2D shell_tex;

static void flush_cb(lv_display_t *disp, const lv_area_t *area, uint8_t *px_map) {
    /* Upload only the dirty sub-rectangle into the raylib-managed GL texture. */
    rlEnableTexture(shell_tex.id);
    glTexSubImage2D(GL_TEXTURE_2D, 0,
                    area->x1, area->y1,
                    area->x2 - area->x1 + 1, area->y2 - area->y1 + 1,
                    GL_RGBA, GL_UNSIGNED_BYTE, px_map);
    rlDisableTexture();
    lv_display_flush_ready(disp);
}

/* Per frame, before raylib drawing: */
/*   lv_tick_inc((uint32_t)(GetFrameTime() * 1000.0f)); */
/*   lv_timer_handler(); */
/*   BeginDrawing(); */
/*     DrawTextureRec(shell_tex, (Rectangle){0, 0, W, H}, (Vector2){0, 0}, WHITE); */
/*   EndDrawing(); */
```

Key implementation details:

- **Color format:** configure `LV_COLOR_DEPTH=32` and match raylib's RGBA8 texture format; verify alpha channel order and byte order before assuming correctness.
- **Partial upload:** LVGL's `flush_cb` already reports dirty `area`s; use `glTexSubImage2D` on those rectangles. A first pass may use whole-frame `UpdateTexture` to validate correctness, then switch to sub-rect uploads for the 60 fps budget.
- **Tick:** drive `lv_tick_inc` from raylib's `GetFrameTime()`; call `lv_timer_handler()` once per frame.
- **Input:** create a `KEYPAD` indev whose `read_cb` translates the existing `input.c` controller state into `LV_KEY_UP/DOWN/LEFT/RIGHT/NEXT/PREV/ENTER/ESC`, and use `lv_gridnav` for 2D focus movement. A/B map to `LV_KEY_ENTER`/`LV_KEY_ESC`.
- **Keep LVGL in software-renderer mode for this smoke test.** Do not enable LVGL's NanoVG/GL draw units in this hybrid; raylib owns GL state. Path 2 revisits this by enabling a GPU draw unit.

---

## Spike Scope

### In Scope (this sprint)

- Vendor LVGL v9.5 under `playos-shell/external/lvgl` and gate it behind a CMake option (e.g. `PLAYOS_SHELL_EXPERIMENTAL_LVGL=ON`).
- Add an experimental screen that registers an LVGL display and renders a small test UI (a grid of buttons/list + a couple of widgets) with a custom or built-in theme. Start with the Path 1 `flush_cb` into a raylib `Texture2D`; re-evaluate against Path 2/3 before doing anything beyond the smoke test.
- Map the existing controller evdev input to an LVGL `KEYPAD` indev with group/gridnav navigation.
- Verify 60 fps, correct colors, partial-upload behaviour, and no blocking I/O in the nested-Wayland dev environment.
- Record a written go/no-go recommendation for a full LVGL shell port.

### Explicitly Out of Scope / Not Planned

- Changing `rcore_playos.c`, the Wayland/EGL lifecycle, the game ABI, or ADR-0006.
- A full port to `LV_USE_WAYLAND` (Path 3) in this spike unless an ADR decision is made and the sprint is re-scoped mid-implementation.
- Buildroot `package/lvgl` packaging (deferred until the port is approved).
- ROG Ally hardware validation (dev environment only).
- Replacing the existing production screens (HOME/LIBRARY/SETTINGS) in this sprint.

---

## Required Repository Changes

| Repo | Required work |
|---|---|
| `playos-shell` | Vendor LVGL v9.5 under `external/lvgl`; add CMake option; add experimental LVGL screen and input mapping (gated) |
| `playos-spec` | Add `Sprint-22.md`; link from `SUMMARY.md`, `sprints/roadmap.md`, and `post-mvp.md` |

---

## Expected Files and Directories

```text
playos-shell/external/lvgl/                 # VENDOR: LVGL v9.5 source
playos-shell/src/screen_lvgl_spike.c        # ADD: experimental LVGL screen (gated)
playos-shell/include/shell.h                # UPDATE: experimental screen enum entry (gated)
playos-shell/CMakeLists.txt                 # UPDATE: PLAYOS_SHELL_EXPERIMENTAL_LVGL option
playos-spec/src/sprints/Sprint-22.md        # ADD: this sprint
playos-spec/src/SUMMARY.md                  # UPDATE: link
playos-spec/src/sprints/roadmap.md          # UPDATE: sprint-plan row
playos-spec/src/post-mvp.md                 # UPDATE: entry
```

---

## Agent Task Breakdown

### Task Status Grid

| Task ID | Task | Primary repo | Status | Notes / evidence |
|---|---|---|---|---|
| S22-T1 | Vendor LVGL and gate behind a CMake option | `playos-shell` | not started | `external/lvgl`, `PLAYOS_SHELL_EXPERIMENTAL_LVGL` |
| S22-T2 | LVGL → raylib texture renderer + test screen (Path 1 smoke test) | `playos-shell` | not started | `flush_cb`, `Texture2D`, `lv_display` |
| S22-T3 | Controller → LVGL keypad/group navigation | `playos-shell` | not started | `input.c`, `lv_indev`, `lv_gridnav` |
| S22-T4 | Verify 60 fps + correctness; write go/no-go | `playos-spec` | not started | nested-Wayland dev env |

### S22-T1 — Vendor LVGL and gate behind a CMake option

- Vendor LVGL v9.5 under `playos-shell/external/lvgl` (mirroring the existing `external/raylib` pattern).
- Add `PLAYOS_SHELL_EXPERIMENTAL_LVGL` (default `OFF`) to `playos-shell/CMakeLists.txt`; when `ON`, compile LVGL and the experimental screen, and link the shell unchanged otherwise.
- Configure LVGL for 32-bit color, no GPU draw unit, and no OS threads; set the tick via the shell loop.

**Done when:** the shell still builds with the option `OFF`, and builds with the option `ON` in the dev environment with LVGL compiled in.

### S22-T2 — LVGL → raylib texture renderer + test screen (Path 1 smoke test)

- Add `screen_lvgl_spike.c` with an `enter/update/draw` triple matching the shell module convention.
- In `enter`, create an LVGL display with a full-screen draw buffer and a `flush_cb` that uploads dirty areas into a raylib `Texture2D`.
- Render a small test UI exercising flexbox/grid, a list or button matrix, and one theme; draw the texture fullscreen in `draw`.

**Done when:** the test screen renders with correct colors through raylib in the nested-Wayland dev environment, with no change to `rcore_playos.c`.

### S22-T3 — Controller → LVGL keypad/group navigation

- Create a `KEYPAD` indev whose `read_cb` maps the existing controller state to `LV_KEY_UP/DOWN/LEFT/RIGHT/NEXT/PREV/ENTER/ESC`.
- Attach `lv_gridnav` (or `lv_group` focus) so D-pad moves focus in two dimensions and A/B activate/back.

**Done when:** D-pad + A/B navigate the test widgets in the dev environment using the existing `input.c` evdev path.

### S22-T4 — Verify 60 fps + correctness; write go/no-go + rendering-path decision

- Confirm the render loop stays within the 16 ms budget using dirty-area sub-rect uploads; fall back to whole-frame upload only for the correctness pass.
- Confirm no blocking I/O on the render thread and that the experimental path is fully gated off in default builds.
- Record the go/no-go recommendation for a full LVGL port **and an explicit rendering-path decision (Path 1 / 2 / 3)** in `Sprint-22.md` (and reflect it in `post-mvp.md`). If the smoke test is clean, the decision may be "jump straight to Path 2 or 3".

**Done when:** the sprint states a measured 60 fps result and a decision-ready go/no-go with rationale, including which rendering path production work should take.

---

## Verification and Evidence

| Evidence | How it is produced |
|---|---|
| Experimental build works | `cmake -B build -DPLAYOS_SHELL_EXPERIMENTAL_LVGL=ON && cmake --build build` in dev env |
| LVGL renders through raylib | Test screen visible in nested Wayland; colors correct (no channel/byte swap) |
| Controller navigation works | D-pad/A/B move focus and activate LVGL widgets |
| 60 fps maintained | Frame-time measured in the dev env; dirty-area upload path used |
| No default-build drift | Build with `PLAYOS_SHELL_EXPERIMENTAL_LVGL=OFF` unchanged |
| Link integrity | `mdbook build` passes |

---

## Acceptance Criteria

- [ ] LVGL v9.5 is vendored under `playos-shell/external/lvgl` and gated behind `PLAYOS_SHELL_EXPERIMENTAL_LVGL` (default `OFF`)
- [ ] A gated experimental screen renders LVGL through a raylib `Texture2D` without changing `rcore_playos.c`
- [ ] Controller D-pad + A/B navigate LVGL widgets via `lv_group`/`lv_gridnav`
- [ ] Colors are correct for the chosen 32-bit format (no swapped channels/bytes)
- [ ] The 60 fps target is met using dirty-area sub-rectangle texture uploads
- [ ] No blocking I/O is introduced on the render thread, and default builds are unchanged
- [ ] A written go/no-go recommendation for a full LVGL port is recorded
- [ ] `SUMMARY.md`, `sprints/roadmap.md`, and `post-mvp.md` are updated
- [ ] `mdbook build` passes

---

## Handoff to Post-MVP

After this sprint:

- The "can LVGL build our shell?" question has a **measured** answer, not just a paper assessment.
- The smoke-test texture blit is proven or rejected as the cheapest correctness check — and a rendering-path decision (Path 1 / 2 / 3) is recorded, so production work can go straight to the GPU-accelerated path when justified.
- A full LVGL port (Path 3, `LV_USE_WAYLAND`) is either recommended for a follow-up sprint (with an ADR), deferred in favour of Path 2, or explicitly ruled out with evidence.

---

## Exit Gate

The spike compiles in the dev environment, renders an LVGL test screen through the existing raylib texture path at 60 fps with controller navigation, and produces a written go/no-go recommendation **plus an explicit rendering-path decision (Path 1 / 2 / 3)** — without touching `rcore_playos.c`, the game ABI, or ADR-0006 (unless Path 3 is chosen, in which case an ADR is required first).

*Previous: [Sprint 21](Sprint-21.md)*
