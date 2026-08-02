# Sprint 4 — AMDGPU and Native DRM/KMS

**Goal:** `playos-compositor` permanently owns the ROG Ally display through the DRM/KMS backend with AMDGPU, GBM, EGL, and Mesa. A hardware-accelerated test client renders to the physical screen.

**Primary Outcome:** The compositor starts on the ROG Ally, initializes AMDGPU via DRM/KMS, and presents a hardware-accelerated test client surface on the built-in display.

**Prerequisites:** Sprint 3 complete — ROG Ally boots, AMDGPU driver loads, `/dev/dri/` is populated.

---

## Key Deliverables

### DRM/KMS Backend in `playos-compositor`

Replace the headless/nested backend with a real DRM backend for the physical device.

**GPU discovery (do not assume `/dev/dri/card0`):**
1. Enumerate all DRM devices via `libdrm`
2. Resolve each to its PCI identity (vendor + device ID)
3. Identify the device connected to the active display connector
4. Validate renderer initialization on the selected device
5. Select the render node (`renderD*`) for EGL context

```c
/* Supported vendor IDs */
#define PCI_VENDOR_AMD   0x1002
#define PCI_VENDOR_INTEL 0x8086  /* deferred until Sprint 13 */
```

Log the selected device, connector, mode, and renderer to `/run/playos/log/compositor.log`.

**DRM/KMS initialization via wlroots:**
- Use `wlr_backend_autocreate()` — it selects DRM on bare metal automatically
- Or use `wlr_drm_backend_create()` directly for explicit device control
- Initialize `wlr_renderer` with GLES2 or GLES3 (GBM path)
- Initialize `wlr_allocator` with GBM
- Enumerate outputs; select the primary built-in display

**Output configuration:**
- Select the preferred mode (native resolution and refresh rate)
- Log: connector name, mode, and format
- Set output scale = 1.0 initially (HiDPI scaling deferred)
- Handle output hotplug events (external display attach/detach) — log only for now

**Renderer:**
- `wlr_renderer` backed by `GBM + EGL + OpenGL ES`
- Mesa RadeonSI for the AMDGPU device
- Verify renderer reports OpenGL ES 3.0 or 3.1

**Fallback diagnostic mode:**
- If DRM/KMS initialization fails, log the failure and attempt SimpleDRM (firmware framebuffer)
- If SimpleDRM also fails, write a diagnostic to serial/log and halt
- This fallback is the recovery graphics path

### Hardware-Accelerated Test Client

Extend the Sprint 2 test client to use OpenGL ES:
- Connect to compositor Wayland socket
- Create an EGL surface via `eglCreateWindowSurface`
- Render a colored gradient with a simple GLES2 shader
- Display: PlayOS name, sprint number, GPU name, resolution, refresh rate
- Animates (e.g., color cycle) to confirm active rendering

### Direct Scanout (Optional — Stretch Goal)

If time permits, implement a basic direct scanout attempt:
- Check if the game surface buffer is compatible with the DRM output plane
- Assign it directly to the plane; fall back to composition if incompatible
- Log scanout decisions

This is an optimization only — correctness is not gated on it.

### `playos-compositor` DRM Configuration

Update `br2-external/configs/playos_rog_ally_defconfig`:
- Ensure `libdrm`, `GBM`, `EGL`, `Mesa` are present in the Buildroot config
- Mesa must include: RadeonSI, GBM, EGL, OpenGL ES (`gallium-drivers=radeonsi`, `platforms=drm,wayland`)
- Validate Mesa is musl-compatible (it is, but verify in practice)

---

## Acceptance Criteria

- [ ] Compositor starts on the ROG Ally using the DRM backend (not headless or nested)
- [ ] AMDGPU DRM device is correctly identified and selected (not hardcoded `/dev/dri/card0`)
- [ ] Compositor log shows: selected GPU, connector, mode, renderer (GLES version)
- [ ] Test client renders a hardware-accelerated surface on the built-in display
- [ ] Mesa reports OpenGL ES 3.0 or 3.1 on AMD
- [ ] Compositor survives: connect USB keyboard, enter BusyBox shell from a second TTY — display is stable
- [ ] If DRM init fails (simulated), compositor logs the failure and enters fallback path
- [ ] QEMU headless path is unchanged — CI still passes

---

## Repositories Primarily Involved

| Repo | Work |
|---|---|
| `playos-compositor` | DRM/KMS backend, GPU discovery, GLES renderer, output config |
| `playos-refdistro` | Mesa config, libdrm, GBM, EGL in Buildroot; ROG Ally defconfig update |
| `playos-spec` | ADR on GPU discovery policy; update graphics architecture notes |

---

## Testing Approach

- Physical ROG Ally required for all DRM/KMS tests
- QEMU CI continues to use headless backend — DRM path tested on device only
- Log validation: parse `/run/playos/log/compositor.log` for expected GPU/mode/renderer lines
- Visual: hardware-accelerated animation visible on Ally screen
- Stress: run compositor for 10 minutes; check for memory leaks or GPU hangs

---

## Exit Gate

`playos-compositor` initializes AMDGPU via DRM/KMS on the ROG Ally and presents a hardware-accelerated test client on the built-in display. GPU is selected by enumeration, not hardcoded path.

*Previous: [Sprint 3](Sprint-3.md) | Next: [Sprint 5](Sprint-5.md)*
