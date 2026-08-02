# Sprint 13 — Intel Expansion

**Goal:** Prove that the PlayOS architecture, compositor, and `playos-platform-api` backend model are portable to Intel graphics hardware. The compositor selects the correct GPU by PCI enumeration (not hardcoded path). A second `libplayos` input/graphics backend compiles and runs on an Intel PC.

**Primary Outcome:** PlayOS boots and runs the full shell + game lifecycle on an Intel-graphics PC (NUC, laptop, or similar). No code path is hardcoded to AMD. The `playos-platform-api` backend abstraction is validated as truly portable.

**Prerequisites:** Sprint 12 complete — AMD implementation complete and hardened.

---

## Key Deliverables

### GPU Discovery Generalization

The compositor already enumerates DRM devices and selects by PCI identity (Sprint 4). Validate that it correctly selects Intel hardware without code changes to the selection logic.

**Supported vendor IDs:**
```c
#define PCI_VENDOR_AMD   0x1002
#define PCI_VENDOR_INTEL 0x8086
```

**Compositor GPU selection fallback order:**
1. Device connected to an active display connector
2. AMD (if multiple GPUs; AMD is primary)
3. Intel
4. First valid DRM device
5. Fatal error if none found

Log the selected vendor, device ID, and device path. No `card0` hardcoding anywhere.

### Intel Kernel Configuration

Create `br2-external/configs/playos_intel_pc_defconfig` starting from a known Intel-compatible configuration.

**Additional kernel modules vs AMD config:**

| Subsystem | Config symbols |
|---|---|
| Intel GPU (Gen 8+) | `CONFIG_DRM_I915` or `CONFIG_DRM_XE` (depending on target HW generation) |
| Intel firmware | `i915/` firmware blobs |
| Intel audio | `CONFIG_SND_HDA_INTEL` + Intel-specific codecs |
| Intel power | `CONFIG_X86_INTEL_PSTATE`, `CONFIG_INTEL_RAPL` |

The AMD-specific configs (`CONFIG_DRM_AMDGPU`, `CONFIG_X86_AMD_PSTATE`) are disabled in the Intel defconfig.

### Mesa Intel Backend

Add to the Buildroot configuration:
- Mesa with `gallium-drivers=iris` (Intel Iris for Gen 9+) or `i965` (older)
- ANV (Intel Vulkan) is deferred to a future Vulkan sprint
- GBM, EGL, OpenGL ES — same as AMD config

Verify: Mesa reports `Mesa ... on Intel ...` when the compositor initializes on Intel hardware.

### `playos-platform-api` — Backend Abstraction

The Platform API backends must be selectable without recompiling the public headers.

**Backend interface (internal to `playos-platform-api/src/`):**

```c
typedef struct {
    const char *name;   /* "amd-evdev", "intel-evdev", etc. */
    int (*init)(void);
    int (*get_controller_state)(PlayOSControllerState *);
    void (*shutdown)(void);
} PlayOSInputBackend;
```

**At runtime:** `playos-platform-api` selects the backend based on `PLAYOS_BACKEND` env var or auto-detection. For input, evdev is hardware-agnostic and should work unchanged. For GPU info queries, select based on PCI vendor in `playos_system.h`.

**Power API — Intel variant:**

| Data | Intel sysfs path |
|---|---|
| CPU temperature | `/sys/class/thermal/thermal_zone*/temp` (find "coretemp") |
| GPU temperature | `/sys/class/drm/card*/device/hwmon/hwmon*/temp1_input` |
| Power profile | `/sys/devices/system/cpu/cpu0/cpufreq/energy_performance_preference` (same interface as AMD P-state) |
| Battery | Same as AMD: `/sys/class/power_supply/BAT*/` |

The `playos_power_request_profile()` implementation is already hardware-agnostic (EPP sysfs interface). Verify it works on Intel.

### New Buildroot Target

Add to `Makefile`:
```
make intel-config    # configure Buildroot for Intel PC target
make intel-build     # build for Intel PC
make intel-usb-image # USB-bootable image for Intel PC
```

### Sample Game Portability Validation

Run all three sample games (`sample-triangle`, `sample-input`, `sample-audio`) on the Intel PC and verify they work identically to AMD:
- Hardware-accelerated rendering (not software fallback)
- Controller input (USB gamepad if no built-in controller)
- Audio output
- System button and lifecycle events

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

## Repositories Primarily Involved

| Repo | Work |
|---|---|
| `playos-compositor` | GPU vendor selection validation, fallback order test |
| `playos-platform-api` | Backend abstraction formalization, Intel power sysfs validation |
| `playos-refdistro` | Intel defconfig, Mesa Iris, Intel firmware, `make intel-*` targets |
| `playos-spec` | Supported hardware matrix, backend portability guide, Intel bring-up notes |

---

## Testing Approach

- Physical Intel PC required for device tests
- Same smoke test checklist as ROG Ally (boot, shell, controller, sample games, lifecycle, audio)
- CI: Intel build must compile cleanly; no runtime tests on Intel hardware in CI (device-only)
- Regression: full ROG Ally smoke test after Intel changes land

---

## Exit Gate

PlayOS runs the full console lifecycle on an Intel-graphics PC. No hardcoded AMD/Intel paths remain in the compositor or `libplayos`. The `playos-platform-api` backend model is validated as portable.

*Previous: [Sprint 12](Sprint-12.md) | Next: [Sprint 14](Sprint-14.md)*
