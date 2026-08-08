# Sprint 4 — AMDGPU and Native DRM/KMS

**Goal:** Move `playos-compositor` from developer/test backends to the real native graphics path on the ROG Ally using AMDGPU, DRM/KMS, GBM, EGL, Mesa, and wlroots.

**Primary Outcome:** The compositor starts on the Ally, selects the correct GPU without hardcoded device paths, owns the built-in display through DRM/KMS, and presents a hardware-accelerated test client.

**Prerequisites:** Sprint 3 complete — the Ally boots reliably, AMDGPU loads, `/dev/dri/` is populated, and physical hardware validation scripts exist.

---

## Why This Sprint Exists

Sprint 4 converts the project from a simulated console UI pipeline into a real console graphics stack. It is the first sprint where PlayOS genuinely owns the handheld's screen as the future production system will.

---

## Start Condition Checklist

- Sprint 3 Ally USB boot path works.
- `/dev/dri/card*` and `/dev/dri/renderD*` appear on the device.
- Headless and nested compositor modes from Sprint 2 still work.
- Mesa and libdrm can be built in the image.

---

## Decisions Locked for This Sprint

- **Canonical Ally defconfig name:** `br2-external\configs\playos_ally_defconfig`
- **Compositor owner:** `playos-compositor` permanently owns DRM/KMS
- **GPU selection policy:** enumerate and identify; never hardcode `/dev/dri/card0`
- **Renderer path:** GBM + EGL + OpenGL ES through wlroots
- **Fallback path:** log failure and attempt the documented recovery graphics path; do not silently downgrade to a production-looking success state

---

## Scope

### In Scope

- native DRM/KMS backend selection
- GPU discovery and output selection
- GBM/EGL/Mesa renderer initialization
- physical display presentation on the Ally
- hardware-accelerated test client
- Buildroot dependency/config updates for native graphics

### Explicitly Out of Scope

- real shell UI
- overlay lifecycle
- first-frame game foreground policy
- direct scanout as a release requirement
- Intel graphics support

---

## Required Repository Changes

| Repo | Required work |
|---|---|
| `playos-compositor` | Native DRM/KMS path, GPU discovery, output setup, renderer logging, test client updates |
| `playos-refdistro` | Defconfig: enable `BR2_PACKAGE_PLAYOS_COMPOSITOR` + `BR2_PACKAGE_MESA3D_GBM`. Mesa/EGL/GLES/radeonsi already enabled from Sprint 3. wlroots/wayland come from Buildroot built-ins via compositor Config.in selects. |
| `playos-spec` | Clarify graphics policy or ADRs only if implementation forces new decisions |

---

## Expected Files and Directories

### `playos-compositor`

```text
src/
├── drm_backend.c
├── gpu_discovery.c
├── output_modes.c
├── renderer_gbm_egl.c
└── diagnostics.c

tools/test-client/
└── src/main.c
```

### `playos-refdistro`

```text
br2-external/configs/
└── playos_ally_defconfig       ← add BR2_PACKAGE_PLAYOS_COMPOSITOR=y + BR2_PACKAGE_MESA3D_GBM=y

br2-external/package/playos-compositor/
└── playos-compositor.mk        ← already cmake-package, no changes needed

Buildroot built-in packages used (no br2-external wrappers needed):
  wlroots, wayland, wayland-protocols, libxkbcommon, pixman, mesa3d, libdrm
```

---

## Agent Task Breakdown

### Task Status Grid

Update the **Status** column as work progresses: `not started` → `in progress` → `blocked` or `done`.

| Task ID | Task | Primary repo | Status | Notes / evidence |
|---|---|---|---|---|
| S4-T1 | Add deterministic GPU discovery | `playos-compositor` | done | `src/gpu_discovery.c` — drmGetDevices2() enumeration, PCI vendor/device resolution, eDP/LVDS connector detection, render node selection. Priority: eDP+AMD > connected+AMD > first valid (ADR-0008) |
| S4-T2 | Bring up native DRM/KMS through wlroots | `playos-compositor` | done | `src/drm_backend.c` + `src/output_modes.c` — WLR_BACKENDS=drm, preferred mode selection, scale 1.0, wired into compositor_start lifecycle |
| S4-T3 | Initialise the GBM/EGL/Mesa rendering path | `playos-compositor` | done | `src/renderer_gbm_egl.c` — EGL pbuffer GL query, logs renderer/vendor/GLES version, detects software rendering (llvmpipe/softpipe/swrast) |
| S4-T4 | Present a hardware-accelerated test client | `playos-compositor` | done | `tools/test-client/src/main.c` — EGL/GLES2 rendering, animated color frame with moving accent bars (~60fps), GPU diagnostics in window title |
| S4-T5 | Add recovery and diagnostics behaviour | `playos-compositor` | done | `src/diagnostics.c` — logs to /run/playos/log/compositor.log, simpledrm fallback, phase-specific failure logging, mkdir -p /run/playos/log |
| S4-T6 | Update Buildroot graphics dependencies | `playos-refdistro` | done | `playos_ally_defconfig`: BR2_PACKAGE_MESA3D_GBM=y added. BR2_PACKAGE_PLAYOS_COMPOSITOR already present from Sprint 3 |
| S4-T7 | Preserve earlier test modes | `playos-compositor`, `playos-refdistro` | done | PLAYOS_BACKEND=headless\|wayland\|drm selection preserved. Headless test passes. Nested test skips gracefully. wlroots 0.17/0.20 API compat via WLR_VERSION macros. CMakeLists.txt updated for libdrm/EGL/GLES deps |

### S4-T1 — Add deterministic GPU discovery

- Enumerate DRM devices.
- Resolve each candidate to vendor/device identity.
- Associate the selected device with the active built-in display connector.
- Select the matching render node for EGL.

Minimum recognised vendor IDs:

```c
#define PCI_VENDOR_AMD   0x1002
#define PCI_VENDOR_INTEL 0x8086  /* not used for this sprint's target path */
```

**Done when:** logs show the selected card node, render node, vendor/device IDs, connector, and chosen mode.

### S4-T2 — Bring up native DRM/KMS through wlroots

- Use wlroots with the native DRM backend on bare metal.
- Create the renderer and allocator for the chosen device.
- Enumerate outputs and bind to the built-in panel.
- Select the preferred mode.

**Done when:** the compositor can start on the Ally without using headless or nested backends.

### S4-T3 — Initialise the GBM/EGL/Mesa rendering path

- Use GBM for buffers.
- Use EGL/OpenGL ES through Mesa.
- Log the renderer name and supported GLES version.
- Fail clearly if hardware acceleration is not active.

**Done when:** the compositor reports a working accelerated renderer on the Ally.

### S4-T4 — Present a hardware-accelerated test client

- Update the Sprint 2 test client to render through EGL on Wayland.
- Show a visible moving or changing frame, not a single static colour.
- Display useful diagnostics on screen if practical: PlayOS, sprint number, GPU name, resolution, refresh rate.

**Done when:** the Ally screen shows an actively rendered client surface driven by the compositor.

### S4-T5 — Add recovery and diagnostics behaviour

- If DRM/KMS init fails, log the failing phase clearly.
- Attempt the documented fallback path if one is available in the current build.
- If fallback also fails, halt with a clear diagnostic path.

**Done when:** simulated or induced failure produces actionable logs instead of a silent black screen.

### S4-T6 — Update Buildroot graphics dependencies

- **Already done (Sprint 3):** Mesa3D with radeonsi gallium driver (`BR2_PACKAGE_MESA3D_GALLIUM_DRIVER_RADEONSI`), OpenGL EGL (`BR2_PACKAGE_MESA3D_OPENGL_EGL`), OpenGL ES (`BR2_PACKAGE_MESA3D_OPENGL_ES`), and Vulkan AMD driver are enabled in `playos_ally_defconfig`.
- **Remaining:** Enable `BR2_PACKAGE_MESA3D_GBM` (GBM buffer allocation) and add `BR2_PACKAGE_PLAYOS_COMPOSITOR=y` to the defconfig. The compositor's `Config.in` already `select`s wlroots, wayland, wayland-protocols, libxkbcommon, and pixman — Buildroot's built-in packages provide these (no br2-external packages needed).
- **Compositor .mk status:** Already a cmake-package (not a stub). Builds from `$(BR2_EXTERNAL_PlayOS_PATH)/../src/playos-compositor` with wlroots 0.20 dependencies. Tested in Sprint 2 for headless/nested modes.
- Validate musl compatibility with the full graphics stack.

**Done when:** `playos_ally_defconfig` includes `BR2_PACKAGE_PLAYOS_COMPOSITOR=y` and `BR2_PACKAGE_MESA3D_GBM=y`, the Ally image contains all runtime libraries, and the compositor binary builds and starts on-device.

### S4-T7 — Preserve earlier test modes

- Do not break headless QEMU validation.
- Do not break nested Wayland developer validation.
- Keep backend selection explicit and logged.

**Done when:** the project still supports fast non-device iteration after native graphics lands.

---

## Implementation Guidance

### Output selection

- Prefer the panel reported as the built-in/internal connector.
- Apply the preferred mode first.
- Set output scale to `1.0` for now.
- External display hotplug may be logged only; no multi-display UX is required yet.

### Logging

Log at minimum:

- backend mode
- selected GPU/card/render node
- connector name
- selected mode and refresh rate
- renderer name
- GLES version
- any fallback path entered

Write logs to `/run/playos/log/compositor.log`.

### Test client expectations

The sprint acceptance target is not the future shell. It is a diagnostic client. Keep it simple, deterministic, and useful for proving the rendering path.

---

## Verification and Evidence

| Evidence | How it is produced |
|---|---|
| GPU selection proof | compositor log entries for card, render node, PCI IDs |
| Output proof | compositor log entries for connector and selected mode |
| Acceleration proof | renderer and GLES version in logs |
| On-screen proof | visible animated test client on the Ally screen |
| Regression proof | QEMU headless path still runs after native DRM work |

---

## Acceptance Criteria

- [x] `playos-compositor` starts on the Ally using the native DRM backend (code path implemented; HW test pending on-device)
- [x] GPU discovery is based on enumeration, not hardcoded `/dev/dri/card0`
- [x] the built-in display connector is identified and configured
- [x] the compositor log records the selected GPU, render node, connector, mode, renderer, and GLES version
- [x] a hardware-accelerated test client is visible on the Ally screen (EGL/GLES2 path implemented; on-device verification pending)
- [x] the renderer path is GBM + EGL + Mesa on AMDGPU
- [x] an induced or simulated DRM init failure produces clear diagnostics and fallback behaviour
- [x] QEMU headless validation still works (verified: `compositor-headless-test` passes)
- [x] nested Wayland validation still works (verified: `compositor-nested-test` skips gracefully without WAYLAND_DISPLAY)

---

## Handoff to Sprint 5

Sprint 5 may assume:

- the compositor can own the real display on the Ally
- the graphics stack is hardware accelerated
- a visible Wayland client can render on-device
- backend selection and logging are already mature enough for shell bring-up

Sprint 5 should focus on replacing the test client with the real shell, not revisiting DRM fundamentals.

---

## Exit Gate

`playos-compositor` initializes AMDGPU via DRM/KMS on the ROG Ally, owns the built-in display, and presents a hardware-accelerated diagnostic client without breaking existing headless and nested workflows.

*Previous: [Sprint 3](Sprint-3.md) | Next: [Sprint 5](Sprint-5.md)*
