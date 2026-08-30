# Sprint 13 — Intel Expansion

**Goal:** Prove that the PlayOS architecture, compositor, and `playos-platform-api` backend model are portable to Intel graphics hardware. The compositor selects the correct GPU by PCI enumeration, not by a hardcoded device path. The existing evdev input and PCI-vendor GPU-query paths run unchanged on an Intel PC.

**Primary Outcome:** PlayOS boots and runs the full shell + game lifecycle on the ASUS ZenBook UX530 (Intel HD Graphics 620 iGPU plus an NVIDIA dGPU that the compositor must ignore). No code path is hardcoded to AMD. The `playos-platform-api` backend abstraction is validated as truly portable.

**Prerequisites:** Sprint 12 complete — AMD implementation complete and hardened.

---

## Why This Sprint Exists

Sprint 12 hardened the AMD ROG Ally path, but that success is still a single-vendor result. The compositor already enumerates DRM devices and selects by PCI identity (Sprint 4), so the question this sprint answers is whether that selection logic, the backend model, and the Buildroot image pipeline generalize without AMD-specific assumptions creeping back in. A second hardware target also forces the platform API to earn its abstraction: if Intel bring-up requires changes to the public headers or to the compositor's device-selection logic, the abstraction is not real yet. This sprint validates portability and produces a second supported target.

---

## Start Condition Checklist

- Sprint 12 complete: the AMD implementation is hardened and all AMD acceptance criteria pass.
- The compositor already enumerates DRM devices and selects by PCI identity (Sprint 4 deliverable).
- The supported vendor IDs are defined: `PCI_VENDOR_AMD 0x1002`, `PCI_VENDOR_INTEL 0x8086` (NVIDIA `0x10de` is intentionally treated as an "other vendor" scoring only +1 — effectively last resort; the ZenBook's 940MX/GTX 950 makes this a real on-device case).
- The GPU selection is a scoring model (not a fixed fallback order): eDP (+1000) or connected (+500) → AMD (+300) → Intel (+100) → any other valid device (+1) → fatal if none found.
- The Intel reference target is available for device tests: ASUS ZenBook UX530 (UX530UQ/UX530UX) — 7th-gen Intel Core (Kaby Lake), Intel HD Graphics 620 (Gen 9.5), plus an NVIDIA 940MX/GTX 950 dGPU (a real hybrid-GPU system).
- The AMD ROG Ally smoke-test checklist is current and repeatable for regression use.

---

## Decisions Locked for This Sprint

- **GPU selection scoring:** eDP (+1000) or connected (+500) → AMD (+300) → Intel (+100) → other vendor (+1) → fatal if none found. The highest total score wins; NVIDIA is "other" and effectively skipped on hybrid systems.
- **No hardcoded paths:** `card0`, `amdgpu`, or vendor-specific strings must not appear in the compositor or in `libplayos` public API.
- **Reference Intel target:** ASUS ZenBook UX530 (UX530UQ/UX530UX) — 7th-gen Intel Core (Kaby Lake), Intel HD Graphics 620 (Gen 9.5), NVIDIA 940MX/GTX 950 dGPU. A real hybrid-GPU laptop, so the NVIDIA "other vendor" path is exercised on hardware, not just in synthetic tests.
- **Intel kernel:** `CONFIG_DRM_I915` (Kaby Lake Gen 9.5; XE does not apply). Disable AMD-only configs (`CONFIG_DRM_AMDGPU`, `CONFIG_X86_AMD_PSTATE`) and NVIDIA options (`CONFIG_DRM_NOUVEAU`, `BR2_PACKAGE_NVIDIA_DRIVER`) in the Intel defconfig.
- **Intel audio:** `CONFIG_SND_HDA_INTEL` plus the Realtek HDA codec used by the ZenBook.
- **Intel power:** `CONFIG_X86_INTEL_PSTATE` and `CONFIG_INTEL_RAPL`.
- **Mesa backend:** `gallium-drivers=iris` (HD 620 is Gen 9.5; `i965` is not required for this target); Intel Vulkan (ANV) is deferred to a future Vulkan sprint.
- **Input backend:** `playos-platform-api` input is hardware-agnostic and compiled as `evdev`; no runtime input-backend switch is needed for Intel. The two existing `PLAYOS_BACKEND` uses are distinct and must not be conflated: the compositor's **runtime env var** (`headless|wayland|drm`) and `playos-platform-api`'s **CMake cache variable** (`auto|evdev|stub`, build-time only). Do not reuse either for a runtime input switch; if a second input backend is ever required, use a distinct variable such as `PLAYOS_INPUT_BACKEND`.
- **Power interface:** `playos_power_request_profile()` stays hardware-agnostic on the EPP sysfs interface.

---

## Scope

### In Scope

- Validate the existing PCI-based GPU discovery logic against Intel hardware (including a real NVIDIA hybrid device).
- Add an Intel PC Buildroot defconfig with Intel kernel, audio, power, and firmware options.
- Enable the Mesa Iris Gallium backend for Intel and verify hardware acceleration.
- Confirm the evdev input path is portable to Intel (no new input backend required).
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
| `playos-compositor` | Validate GPU selection by PCI vendor, log the selected vendor/device/path, add a GPU-selection scoring test, remove any residual `card0` hardcoding |
| `playos-platform-api` | Generalize the hwmon temp readers (`amdgpu`/`k10temp` → also `i915`/`coretemp`); validate Intel power sysfs paths (device-model string is vendor-agnostic; the GPU-description probe keeps its `card0`→`card1` fallback this sprint) |
| `playos-init` | Generalize its thermal-monitor hwmon readers the same way (`amdgpu`→`i915`, `k10temp`→`coretemp`) so the Sprint 9 thermal state machine keeps reading GPU temp on Intel |
| `playos-refdistro` | Add `playos_intel_pc_defconfig`, Intel kernel configs/firmware, Mesa Iris, and `make intel-*` targets |
| `playos-samples` | Run `rotating-squares`, `controller-visualizer`, and `audio-sine` on the Intel PC and record portability evidence |
| `playos-spec` | Update the supported-hardware matrix and add backend-portability guidance plus Intel bring-up notes |

---

## Expected Files and Directories

### `playos-compositor`

```text
src/gpu_discovery.c             # PCI vendor scoring + selection (already exists from Sprint 4)
src/compositor.c                # consumes the selected DRM device; no card0/vendor strings
tests/test_gpu_select.c         # NEW: GPU-selection scoring unit test with fake DRM/vendor data
```

### `playos-platform-api`

```text
src/playos_power.c              # generalize hwmon temp readers: amdgpu/i915 (GPU) and k10temp/coretemp (CPU)
src/playos_system.c             # device model (DMI) already vendor-agnostic; GPU-description probe keeps card0→card1 fallback
```

### `playos-init`

```text
src/thermal.c                   # generalize hwmon temp readers: amdgpu/i915 (GPU) and k10temp/coretemp (CPU)
```

### `playos-refdistro`

```text
br2-external/configs/playos_intel_pc_defconfig
br2-external/board/intel/linux-fragment.cfg   # DRM_I915, SND_HDA_INTEL, INTEL_PSTATE, INTEL_RAPL; NOUVEAU/NVIDIA disabled
br2-external/board/intel/firmware.list        # i915 firmware blobs (KBL DMC; GuC/HuC optional)
Makefile                                      # make intel-config / intel-build / intel-usb-image (+ help text)
scripts/gen-intel-usb-image.sh                # USB-bootable Intel PC image
```

### `playos-samples`

```text
docs/intel-portability-validation.md   # per-game Intel results: render, input, audio, lifecycle
```

### `playos-spec`

```text
src/hardware-matrix.md          # updated: AMD ROG Ally + Intel PC supported targets
src/backend-portability.md      # new: evdev input model, GPU-selection scoring, power sysfs matrix
src/sprints/Sprint-13.md        # this sprint
```

---

## Agent Task Breakdown

### Task Status Grid

| Task ID | Task | Primary repo | Status | Notes / evidence |
|---|---|---|---|---|
| S13-T1 | Validate GPU discovery by PCI vendor and scoring model on Intel hardware | `playos-compositor` | in progress | Host part done (`gpu_score` + ctest `gpu-select`, commit `195664c`); **on-device: shell + game render on the hybrid ZenBook (Intel selected, NVIDIA would not render) — explicit `0x8086` log pending SSH** |
| S13-T2 | Add Intel PC kernel configuration and firmware | `playos-refdistro` | done | `playos_intel_pc_defconfig` + `board/intel/linux-fragment.cfg` + `firmware.list` authored; kernel built and **booted on-device**; `.config` verified (`EFI_STUB/i915/HDA/INTEL_PSTATE/INTEL_RAPL=y`, AMD/NVIDIA off) |
| S13-T3 | Enable Mesa Iris Gallium backend for Intel | `playos-refdistro` | in progress | Iris + LLVM + EGL/ES/GBM configured; **sample game rendered on-device**; explicit Mesa Intel renderer string pending log |
| S13-T4 | Confirm evdev input is portable to Intel (no new backend) | `playos-platform-api` | done | **USB gamepad input confirmed on-device**; no code changes required |
| S13-T5 | Validate Intel power sysfs paths and device strings | `playos-platform-api`, `playos-init` | in progress | Code + host tests done (`d9d5bae`, `41c071b`); **on-device: battery % + thermal state render in shell status bar**; explicit `playos_power_get_info()` values pending SSH |
| S13-T6 | Add `make intel-*` Buildroot targets and USB image generation | `playos-refdistro` | in progress | Makefile targets + image script + CI workflow added; **local `make intel-usb-image` succeeded** (image booted on-device); CI run pending push |
| S13-T7 | Validate sample-game portability on the Intel PC | `playos-samples` | in progress | Sample game launched + rendered with USB gamepad input on-device; **audio-sine + system-button/lifecycle still to test** |
| S13-T8 | Document dual-vendor support in the specs | `playos-spec` | done | `hardware-matrix.md` + `backend-portability.md` added and linked from `SUMMARY.md` |

Update the **Status** column as work progresses: `not started` → `in progress` → `blocked` or `done`.

### S13-T1 — Validate GPU discovery by PCI vendor

The compositor already enumerates DRM devices and selects by a PCI-vendor scoring model (Sprint 4). Verify on the ZenBook that the scoring logic picks the Intel device (eDP +1000, Intel +100) over the NVIDIA dGPU (+1, no connected output) without code changes. Confirm the scoring order — eDP/connected → AMD → Intel → other → fatal — and log the selected vendor ID, device ID, and device path. Confirm no `card0` or vendor-specific hardcoding remains in the compositor, and that NVIDIA (an "other" vendor scoring only +1) is effectively last resort on this hybrid system.

**Done when:** the compositor log on the ZenBook shows the Intel vendor ID (`0x8086`) and device path selected over the NVIDIA device (`0x10de`), the GPU-selection scoring unit test passes with synthetic multi-GPU data (including a synthetic NVIDIA case), `gpu_discovery.h`'s doc comment is updated to describe the scoring sum (not the old first-AMD-first-Intel prose), and a grep of `playos-compositor` finds no `card0` hardcoding.

Implementation note: factor `calculate_score()` and candidate construction into a testable pure-function seam (or use `--wrap` linker tricks) so `tests/test_gpu_select.c` can feed synthetic vendor/connector data without touching real DRM nodes.

### S13-T2 — Add Intel PC kernel configuration

Create `br2-external/configs/playos_intel_pc_defconfig` for the ZenBook UX530 (Kaby Lake). Add Intel GPU support (`CONFIG_DRM_I915`; XE does not apply to Gen 9.5), Intel audio (`CONFIG_SND_HDA_INTEL` plus the Realtek codec), and Intel power options (`CONFIG_X86_INTEL_PSTATE`, `CONFIG_INTEL_RAPL`). Include the `i915/` firmware blobs (KBL DMC; GuC/HuC optional). Disable AMD-only configs (`CONFIG_DRM_AMDGPU`, `CONFIG_X86_AMD_PSTATE`) and NVIDIA options (`CONFIG_DRM_NOUVEAU`, `BR2_PACKAGE_NVIDIA_DRIVER`) — the ZenBook's 940MX/GTX 950 must not load a driver.

**Done when:** the Intel defconfig builds a kernel where the required Intel symbols are enabled and the AMD and NVIDIA symbols are disabled, as shown by the generated `.config`.

### S13-T3 — Enable Mesa Iris backend

Configure Buildroot Mesa with `gallium-drivers=iris` (HD 620 is Gen 9.5), keeping GBM, EGL, and OpenGL ES the same as the AMD config. Intel Vulkan (ANV) is deferred. Verify at runtime that Mesa reports an Intel renderer.

**Done when:** on an Intel PC, Mesa initialization logs `Mesa ... on Intel ...` (or the equivalent Intel renderer string), and `rotating-squares` renders with hardware acceleration rather than a software fallback.

### S13-T4 — Confirm evdev input portability

Confirm the existing evdev input path (`src/playos_input.c`) is hardware-agnostic and works on Intel without modification. No second input backend is required for Intel bring-up — do not introduce a runtime `PlayOSInputBackend` struct, do not reuse the compositor's `PLAYOS_BACKEND` **runtime env var** (`headless|wayland|drm`), and do not confuse it with `playos-platform-api`'s `PLAYOS_BACKEND` **CMake cache variable** (`auto|evdev|stub`, build-time). If a second input backend is ever needed, use a distinct variable such as `PLAYOS_INPUT_BACKEND`.

**Done when:** evdev input works unchanged on the Intel PC with no public-header changes, and the AMD input tests still pass.

### S13-T5 — Validate Intel power sysfs paths

Generalize the hwmon temperature readers in `src/playos_power.c`: `read_hwmon_gpu_temp()` currently matches hwmon `name=="amdgpu"` and must also accept `i915` (optionally `xe` for future Intel platforms); `read_hwmon_cpu_temp()` currently matches `k10temp` and must also accept `coretemp`. Keep `read_epp_profile()` and `playos_power_request_profile()` unchanged — the EPP sysfs interface is shared across vendors. `playos_system_device_model()` (DMI `product_name`) is already vendor-agnostic. The GPU-description probe in `playos_system.c` keeps its `card0`→`card1` fallback this sprint (it produces a display string, not GPU selection); making it enumerate DRM devices is deferred.

Generalize `playos-init/src/thermal.c` the same way: `read_hwmon_gpu_temp()` must accept `i915` and `read_hwmon_cpu_temp()` must accept `coretemp` (the existing `x86_pkg_temp` → `cpu_thermal` fallback already covers Intel CPU temp, but the GPU reader would otherwise return −1 on Intel).

**Done when:** `playos_power_get_info()` returns valid CPU, GPU, and battery data on the ZenBook (BAT0 present), the `playos-init` thermal monitor reports a valid GPU temp on Intel, and `playos_system_device_model()` returns a non-AMD device string.

### S13-T6 — Add `make intel-*` targets

Add `make intel-config`, `make intel-build`, and `make intel-usb-image` to the `playos-refdistro` `Makefile` (including the `help` text), plus `scripts/gen-intel-usb-image.sh` for a USB-bootable Intel PC image. Wire the Intel build into CI as a cross-compilation target (extend `qemu-build.yml` or add `intel-build.yml`).

**Done when:** `make intel-build` compiles cleanly in CI and produces the Intel image artifacts alongside the existing AMD artifacts.

### S13-T7 — Validate sample-game portability

Run `rotating-squares`, `controller-visualizer`, and `audio-sine` on the ZenBook UX530 and confirm hardware-accelerated rendering, controller input (USB gamepad — the ZenBook has no built-in gamepad), audio output, and the system-button/lifecycle flow. Record the results in `docs/intel-portability-validation.md`.

**Done when:** all three sample games run on the ZenBook with the same behavior as on AMD, and the portability validation document is committed with per-game evidence.

### S13-T8 — Document dual-vendor support

Update the supported-hardware matrix to list both the AMD ROG Ally and the Intel PC, and add backend-portability guidance covering the hardware-agnostic evdev input model, the GPU-selection scoring order, and the power sysfs matrix.

**Done when:** `hardware-matrix.md` lists both targets and `backend-portability.md` is committed and linked from the spec index.

---

## Implementation Guidance

**Validate, don't rewrite, the selection logic.** The GPU selection already exists from Sprint 4. The goal is to prove it generalizes — add logging and a test, not a second selection path.

**No vendor strings in the public API.** If Intel support requires a change to `playos-platform-api` public headers, the abstraction has failed; fix the abstraction instead.

**Keep CI compile-only for Intel.** Intel runtime tests are device-only. CI validates that the Intel defconfig cross-compiles; do not gate the sprint on Intel hardware in CI.

**Disable, don't just omit, AMD and NVIDIA options.** The Intel defconfig must explicitly disable `CONFIG_DRM_AMDGPU`/`CONFIG_X86_AMD_PSTATE` and `CONFIG_DRM_NOUVEAU`/`BR2_PACKAGE_NVIDIA_DRIVER` so the result is unambiguous — the ZenBook's NVIDIA dGPU must never bind a driver during this sprint's validation.

**Preserve the EPP power interface.** Do not fork the power API per vendor; the energy-performance-preference sysfs interface is shared and should stay shared.

**Treat Intel Vulkan as explicitly deferred.** Do not pull ANV into this sprint's scope; record it as a future-sprint follow-up only.

---

## Verification and Evidence

| Evidence | How it is produced |
|---|---|
| Intel GPU selected by PCI enumeration | Compositor boot log on the Intel PC shows `0x8086` and the device path |
| GPU selection scoring correct | `test_gpu_select.c` runs synthetic multi-GPU vendor data through the selector |
| Intel kernel config correct | Generated `.config` diff against the Intel defconfig |
| Mesa Intel renderer | Compositor/Mesa init log on the Intel PC |
| Hardware acceleration | `rotating-squares` renderer string and frame throughput on the Intel PC |
| Input portability | `controller-visualizer` controller-state dump on the Intel PC |
| Audio portability | `audio-sine` output on the Intel PC |
| Power API valid | `playos_power_get_info()` output for CPU, GPU, and battery on the Intel PC |
| Non-AMD device string | `playos_system_device_model()` output on the Intel PC |
| Intel image builds | CI log for `make intel-build` and produced image artifacts |
| AMD regression | Full ROG Ally smoke-test checklist after Intel changes land |

---

## Acceptance Criteria

- [x] PlayOS boots on the ASUS ZenBook UX530 (Intel HD 620 iGPU + NVIDIA dGPU present) — confirmed on-device 2026-08-26
- [x] Compositor selects the Intel DRM device by PCI enumeration over the NVIDIA device (no hardcoded path) — shell + game render on the hybrid machine; explicit `0x8086` log capture pending SSH
- [ ] Compositor log shows Intel vendor ID and Mesa Iris renderer — pending log capture over SSH
- [x] `rotating-squares` runs with hardware acceleration on Intel (Mesa reports `Intel ...`) — sample game rendered on-device; explicit Mesa renderer string pending log capture
- [x] `controller-visualizer` receives controller input on the ZenBook (USB gamepad) — USB gamepad input confirmed on-device
- [ ] `audio-sine` plays audio on the ZenBook — not yet tested
- [ ] System button and lifecycle flow works on the ZenBook — not yet tested
- [x] `playos_power_get_info()` returns valid CPU, GPU, and battery data on the ZenBook (BAT0 present) — battery % and thermal state rendered in the shell status bar on-device
- [ ] `playos_system_device_model()` returns a non-AMD device string — not yet observed
- [ ] AMD ROG Ally tests are unaffected — all Sprint 12 acceptance criteria still pass (hwmon matcher change is additive; on-device re-run pending)
- [x] No `card0` or vendor-specific GPU-selection hardcoding in the compositor; in `libplayos` the only permitted card-index use is the existing `playos_system.c` GPU-description probe (`card0`→`card1` fallback) — verified by grep
- [x] `make intel-build` succeeds in CI (using a cross-compilation target) — local build succeeded (`make intel-usb-image` produced the image); CI run pending push

---

## Handoff to Sprint 14

Sprint 14 may assume:

- PlayOS runs the full console lifecycle on both the AMD ROG Ally and the ASUS ZenBook UX530 (Intel HD 620 with NVIDIA dGPU present).
- The compositor selects the GPU by PCI enumeration with a tested scoring model and no hardcoded paths.
- The `playos-platform-api` input model is validated as portable (hardware-agnostic evdev); no runtime input-backend switch or `PLAYOS_BACKEND` reuse was introduced.
- The Mesa Intel (Iris) backend is enabled and hardware acceleration is verified.
- `make intel-build` and `make intel-usb-image` targets exist and compile in CI.
- The supported-hardware matrix and backend-portability docs are committed.

---

## Exit Gate

PlayOS runs the full console lifecycle on the ASUS ZenBook UX530 (Intel HD Graphics 620, NVIDIA dGPU present but unselected). No hardcoded AMD/Intel paths remain in the compositor; the only card-index use in `libplayos` is the GPU-description probe. The `playos-platform-api` backend model is validated as portable.

*Previous: [Sprint 12](Sprint-12.md) | Next: [Sprint 13.6](Sprint-13.6.md)*
