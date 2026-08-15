# Sprint 13 — Intel Expansion

**Goal:** Prove that the PlayOS architecture, compositor, and `playos-platform-api` backend model are portable to Intel graphics hardware. The compositor selects the correct GPU by PCI enumeration, not by a hardcoded device path. A second `libplayos` input/graphics backend compiles and runs on an Intel PC.

**Primary Outcome:** PlayOS boots and runs the full shell + game lifecycle on an Intel-graphics PC (NUC, laptop, or similar). No code path is hardcoded to AMD. The `playos-platform-api` backend abstraction is validated as truly portable.

**Prerequisites:** Sprint 12 complete — AMD implementation complete and hardened.

---

## Why This Sprint Exists

Sprint 12 hardened the AMD ROG Ally path, but that success is still a single-vendor result. The compositor already enumerates DRM devices and selects by PCI identity (Sprint 4), so the question this sprint answers is whether that selection logic, the backend model, and the Buildroot image pipeline generalize without AMD-specific assumptions creeping back in. A second hardware target also forces the platform API to earn its abstraction: if Intel bring-up requires changes to the public headers or to the compositor's device-selection logic, the abstraction is not real yet. This sprint validates portability and produces a second supported target.

---

## Start Condition Checklist

- Sprint 12 complete: the AMD implementation is hardened and all AMD acceptance criteria pass.
- The compositor already enumerates DRM devices and selects by PCI identity (Sprint 4 deliverable).
- The supported vendor IDs are defined: `PCI_VENDOR_AMD 0x1002`, `PCI_VENDOR_INTEL 0x8086`.
- The GPU selection fallback order is documented and implemented: active connector → AMD → Intel → first valid DRM device → fatal.
- An Intel-graphics PC (NUC, laptop, or test machine) is available for device tests.
- The AMD ROG Ally smoke-test checklist is current and repeatable for regression use.

---

## Decisions Locked for This Sprint

- **GPU selection fallback order:** active display connector → AMD (primary when multiple GPUs) → Intel → first valid DRM device → fatal if none found.
- **No hardcoded paths:** `card0`, `amdgpu`, or vendor-specific strings must not appear in the compositor or in `libplayos` public API.
- **Intel kernel:** use `CONFIG_DRM_I915` or `CONFIG_DRM_XE` depending on target hardware generation; disable AMD-only configs in the Intel defconfig.
- **Intel audio:** `CONFIG_SND_HDA_INTEL` plus Intel-specific codecs.
- **Intel power:** `CONFIG_X86_INTEL_PSTATE` and `CONFIG_INTEL_RAPL`.
- **Mesa backend:** `gallium-drivers=iris` for Gen 9+ (`i965` for older); Intel Vulkan (ANV) is deferred to a future Vulkan sprint.
- **Backend selection:** `playos-platform-api` selects its backend at runtime via the `PLAYOS_BACKEND` environment variable, with a hardware-agnostic evdev input path and a PCI-vendor-based GPU query path.
- **Power interface:** `playos_power_request_profile()` stays hardware-agnostic on the EPP sysfs interface.

---

## Scope

### In Scope

- Validate the existing PCI-based GPU discovery logic against Intel hardware.
- Add an Intel PC Buildroot defconfig with Intel kernel, audio, power, and firmware options.
- Enable the Mesa Iris Gallium backend for Intel and verify hardware acceleration.
- Formalize the internal `PlayOSInputBackend` abstraction and `PLAYOS_BACKEND` selection.
- Validate `playos_power_get_info()` and profile requests against Intel sysfs paths.
- Add `make intel-config`, `make intel-build`, and `make intel-usb-image` targets.
- Validate all three sample games on the Intel PC.
- Document the dual-vendor support matrix and backend portability guidance.

### Explicitly Out of Scope

- Intel Vulkan (ANV) — deferred to a future Vulkan sprint.
- Runtime testing of Intel hardware in CI — Intel device tests are device-only; CI covers cross-compilation.
- Automatic backend auto-detection beyond the documented env-var and PCI-vendor paths.
- Multi-GPU simultaneous rendering (one active GPU is selected; the other is ignored).

---

## Required Repository Changes

| Repo | Required work |
|---|---|
| `playos-compositor` | Validate GPU selection by PCI vendor, log the selected vendor/device/path, add a fallback-order test, remove any residual `card0` hardcoding |
| `playos-platform-api` | Formalize `PlayOSInputBackend` and `PLAYOS_BACKEND` dispatch; validate Intel power sysfs paths and non-AMD device strings |
| `playos-refdistro` | Add `playos_intel_pc_defconfig`, Intel kernel configs/firmware, Mesa Iris, and `make intel-*` targets |
| `playos-samples` | Run `sample-triangle`, `sample-input`, and `sample-audio` on the Intel PC and record portability evidence |
| `playos-spec` | Update the supported-hardware matrix and add backend-portability guidance plus Intel bring-up notes |

---

## Expected Files and Directories

### `playos-compositor`

```text
src/gpu_select.c                # PCI vendor selection, fallback order, selected-vendor logging
src/compositor.c                # consumes the selected DRM device; no card0/vendor strings
tests/test_gpu_select.c         # fallback-order unit test with fake DRM/vendor data
```

### `playos-platform-api`

```text
src/input_backend.c             # PlayOSInputBackend dispatch via PLAYOS_BACKEND env var
src/power_intel.c               # Intel power sysfs queries (coretemp, GPU hwmon, EPP)
src/playos_system.c             # device-model string: non-AMD value on Intel targets
```

### `playos-refdistro`

```text
br2-external/configs/playos_intel_pc_defconfig
br2-external/board/intel/linux-fragment.cfg   # DRM_I915/XE, SND_HDA_INTEL, INTEL_PSTATE, INTEL_RAPL
br2-external/board/intel/firmware.list        # i915 firmware blobs
Makefile                                      # make intel-config / intel-build / intel-usb-image
gen-intel-usb-image.sh                        # USB-bootable Intel PC image
```

### `playos-samples`

```text
docs/intel-portability-validation.md   # per-game Intel results: render, input, audio, lifecycle
```

### `playos-spec`

```text
src/hardware-matrix.md          # updated: AMD ROG Ally + Intel PC supported targets
src/backend-portability.md      # new: backend model, PLAYOS_BACKEND, power sysfs matrix
src/sprints/Sprint-13.md        # this sprint
```

---

## Agent Task Breakdown

### Task Status Grid

| Task ID | Task | Primary repo | Status | Notes / evidence |
|---|---|---|---|---|
| S13-T1 | Validate GPU discovery by PCI vendor and fallback order on Intel hardware | `playos-compositor` | not started | |
| S13-T2 | Add Intel PC kernel configuration and firmware | `playos-refdistro` | not started | |
| S13-T3 | Enable Mesa Iris Gallium backend for Intel | `playos-refdistro` | not started | |
| S13-T4 | Formalize `PlayOSInputBackend` and `PLAYOS_BACKEND` dispatch | `playos-platform-api` | not started | |
| S13-T5 | Validate Intel power sysfs paths and device strings | `playos-platform-api` | not started | |
| S13-T6 | Add `make intel-*` Buildroot targets and USB image generation | `playos-refdistro` | not started | |
| S13-T7 | Validate sample-game portability on the Intel PC | `playos-samples` | not started | |
| S13-T8 | Document dual-vendor support in the specs | `playos-spec` | not started | |

Update the **Status** column as work progresses: `not started` → `in progress` → `blocked` or `done`.

### S13-T1 — Validate GPU discovery by PCI vendor

The compositor already enumerates DRM devices and selects by PCI identity (Sprint 4). Verify on Intel hardware that the selection logic picks the Intel device without code changes. Enforce the fallback order — active connector → AMD → Intel → first valid DRM device → fatal — and log the selected vendor ID, device ID, and device path. Confirm no `card0` or vendor-specific hardcoding remains.

**Done when:** the compositor log shows the Intel vendor ID (`0x8086`) and device path on an Intel PC, the fallback-order unit test passes with synthetic multi-GPU data, and a grep of `playos-compositor` finds no `card0` hardcoding.

### S13-T2 — Add Intel PC kernel configuration

Create `br2-external/configs/playos_intel_pc_defconfig` from a known Intel-compatible configuration. Add Intel GPU support (`CONFIG_DRM_I915` or `CONFIG_DRM_XE` depending on target generation), Intel audio (`CONFIG_SND_HDA_INTEL` plus codecs), and Intel power options (`CONFIG_X86_INTEL_PSTATE`, `CONFIG_INTEL_RAPL`). Include the `i915/` firmware blobs. Disable AMD-only configs (`CONFIG_DRM_AMDGPU`, `CONFIG_X86_AMD_PSTATE`) in the Intel defconfig.

**Done when:** the Intel defconfig builds a kernel where the required Intel config symbols are enabled and the AMD-only symbols are disabled, as shown by the generated `.config`.

### S13-T3 — Enable Mesa Iris backend

Configure Buildroot Mesa with `gallium-drivers=iris` for Intel Gen 9+ (or `i965` for older), keeping GBM, EGL, and OpenGL ES the same as the AMD config. Intel Vulkan (ANV) is deferred. Verify at runtime that Mesa reports an Intel renderer.

**Done when:** on an Intel PC, Mesa initialization logs `Mesa ... on Intel ...` (or the equivalent Intel renderer string), and `sample-triangle` renders with hardware acceleration rather than a software fallback.

### S13-T4 — Formalize `PlayOSInputBackend`

Define the internal backend struct in `playos-platform-api/src/` with `name`, `init`, `get_controller_state`, and `shutdown` members, and dispatch at runtime from the `PLAYOS_BACKEND` environment variable. The evdev input path remains hardware-agnostic and works unchanged; GPU info queries select by PCI vendor via `playos_system.h`. Public headers must not change to support the second backend.

**Done when:** setting `PLAYOS_BACKEND` selects the requested backend without recompiling public headers, and the AMD backend continues to pass its existing tests unchanged.

### S13-T5 — Validate Intel power sysfs paths

Verify the Intel power sysfs paths: CPU temperature via the `coretemp` thermal zone, GPU temperature via `/sys/class/drm/card*/device/hwmon/hwmon*/temp1_input`, power profile via `/sys/devices/system/cpu/cpu0/cpufreq/energy_performance_preference` (the same EPP interface as AMD P-state), and battery via `/sys/class/power_supply/BAT*/`. Confirm `playos_power_request_profile()` works on Intel without modification.

**Done when:** `playos_power_get_info()` returns valid CPU and battery data on the Intel PC, and `playos_system_device_model()` returns a non-AMD device string.

### S13-T6 — Add `make intel-*` targets

Add `make intel-config`, `make intel-build`, and `make intel-usb-image` to the `playos-refdistro` `Makefile`, plus a `gen-intel-usb-image.sh` for a USB-bootable Intel PC image. Wire the Intel build into CI as a cross-compilation target.

**Done when:** `make intel-build` compiles cleanly in CI and produces the Intel image artifacts alongside the existing AMD artifacts.

### S13-T7 — Validate sample-game portability

Run `sample-triangle`, `sample-input`, and `sample-audio` on the Intel PC and confirm hardware-accelerated rendering, controller input (USB gamepad if no built-in controller), audio output, and the system-button/lifecycle flow. Record the results in `docs/intel-portability-validation.md`.

**Done when:** all three sample games run on the Intel PC with the same behavior as on AMD, and the portability validation document is committed with per-game evidence.

### S13-T8 — Document dual-vendor support

Update the supported-hardware matrix to list both the AMD ROG Ally and the Intel PC, and add backend-portability guidance covering the `PlayOSInputBackend` model, `PLAYOS_BACKEND`, GPU selection fallback order, and the power sysfs matrix.

**Done when:** `hardware-matrix.md` lists both targets and `backend-portability.md` is committed and linked from the spec index.

---

## Implementation Guidance

**Validate, don't rewrite, the selection logic.** The GPU selection already exists from Sprint 4. The goal is to prove it generalizes — add logging and a test, not a second selection path.

**No vendor strings in the public API.** If Intel support requires a change to `playos-platform-api` public headers, the abstraction has failed; fix the abstraction instead.

**Keep CI compile-only for Intel.** Intel runtime tests are device-only. CI validates that the Intel defconfig cross-compiles; do not gate the sprint on Intel hardware in CI.

**Disable, don't just omit, AMD options.** The Intel defconfig must explicitly disable `CONFIG_DRM_AMDGPU` and `CONFIG_X86_AMD_PSTATE` so the result is unambiguous.

**Preserve the EPP power interface.** Do not fork the power API per vendor; the energy-performance-preference sysfs interface is shared and should stay shared.

**Treat Intel Vulkan as explicitly deferred.** Do not pull ANV into this sprint's scope; record it as a future-sprint follow-up only.

---

## Verification and Evidence

| Evidence | How it is produced |
|---|---|
| Intel GPU selected by PCI enumeration | Compositor boot log on the Intel PC shows `0x8086` and the device path |
| Fallback order correct | `test_gpu_select.c` runs synthetic multi-GPU vendor data through the selector |
| Intel kernel config correct | Generated `.config` diff against the Intel defconfig |
| Mesa Intel renderer | Compositor/Mesa init log on the Intel PC |
| Hardware acceleration | `sample-triangle` renderer string and frame throughput on the Intel PC |
| Input portability | `sample-input` controller-state dump on the Intel PC |
| Audio portability | `sample-audio` output on the Intel PC |
| Power API valid | `playos_power_get_info()` output for CPU, GPU, and battery on the Intel PC |
| Non-AMD device string | `playos_system_device_model()` output on the Intel PC |
| Intel image builds | CI log for `make intel-build` and produced image artifacts |
| AMD regression | Full ROG Ally smoke-test checklist after Intel changes land |

---

## Acceptance Criteria

- [ ] PlayOS boots on an Intel-graphics PC (NUC, laptop, or test machine)
- [ ] Compositor selects the Intel DRM device by PCI enumeration (no hardcoded path)
- [ ] Compositor log shows Intel vendor ID and Mesa Iris (or i965) renderer
- [ ] `sample-triangle` runs with hardware acceleration on Intel (Mesa reports `Intel ...`)
- [ ] `sample-input` receives controller input on Intel PC (USB gamepad or built-in)
- [ ] `sample-audio` plays audio on Intel PC
- [ ] System button and lifecycle flow works on Intel PC
- [ ] `playos_power_get_info()` returns valid CPU and battery data on Intel
- [ ] `playos_system_device_model()` returns a non-AMD device string
- [ ] AMD ROG Ally tests are unaffected — all Sprint 12 acceptance criteria still pass
- [ ] No `card0`, `amdgpu`, or AMD-specific hardcoded strings in compositor or `libplayos` public API
- [ ] `make intel-build` succeeds in CI (using a cross-compilation target)

---

## Handoff to Sprint 14

Sprint 14 may assume:

- PlayOS runs the full console lifecycle on both the AMD ROG Ally and an Intel PC.
- The compositor selects the GPU by PCI enumeration with a tested fallback order and no hardcoded paths.
- The `playos-platform-api` backend model is validated as portable via `PlayOSInputBackend` and `PLAYOS_BACKEND`.
- The Mesa Intel (Iris) backend is enabled and hardware acceleration is verified.
- `make intel-build` and `make intel-usb-image` targets exist and compile in CI.
- The supported-hardware matrix and backend-portability docs are committed.

---

## Exit Gate

PlayOS runs the full console lifecycle on an Intel-graphics PC. No hardcoded AMD/Intel paths remain in the compositor or `libplayos`. The `playos-platform-api` backend model is validated as portable.

*Previous: [Sprint 12](Sprint-12.md) | Next: [Sprint 14](Sprint-14.md)*
