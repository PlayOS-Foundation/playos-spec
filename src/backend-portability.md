# Backend Portability

> Last updated: 2026-08-24 (Sprint 13). This document records how the platform
> API and compositor stay vendor-agnostic across AMD and Intel targets.

## Input backend

- `playos-platform-api` input is hardware-agnostic **evdev**. It is selected at
  **build time** by the CMake cache variable `PLAYOS_BACKEND` (`auto|evdev|stub`);
  the ROG Ally and Intel PC images both build it as `evdev`. No runtime input
  switch exists and none is needed for Intel.
- The compositor's `PLAYOS_BACKEND` is a **runtime environment variable**
  (`headless|wayland|drm`) and is unrelated to the platform-api CMake variable.
  Do not conflate the two.
- If a second input backend is ever required, introduce a distinct variable such
  as `PLAYOS_INPUT_BACKEND` rather than overloading either existing use.

## GPU selection (compositor)

`playos-compositor` enumerates DRM devices with `drmGetDevices2()` and scores
each candidate (ADR-0008, implemented in `src/gpu_score.c` since Sprint 13):

| Signal | Score |
|---|---|
| eDP (internal panel) | +1000 |
| connected output (non-eDP) | +500 |
| PCI vendor AMD | +300 |
| PCI vendor Intel | +100 |
| any other vendor (incl. NVIDIA) | +1 |

Highest total wins; ties resolve to the earliest candidate. NVIDIA therefore is
effectively last resort on hybrid systems — on the ZenBook UX530 the Intel iGPU
(eDP + Intel = 1100) always beats the NVIDIA dGPU (+1). The scoring model is
unit-tested with synthetic data (`tests/test_gpu_select.c`, ctest `gpu-select`)
without touching real DRM nodes. No fixed `/dev/dri/card*` paths exist in the
compositor.

## Power / thermal sysfs matrix

| Reading | Path | AMD | Intel |
|---|---|---|---|
| Battery % / status | `/sys/class/power_supply/BAT0/…` | BAT0 | BAT0 (ZenBook laptop) |
| CPU package temp | `/sys/class/thermal/thermal_zone*/type` = `x86_pkg_temp` → `cpu_thermal` fallback | ✓ | ✓ |
| CPU hwmon temp | `/sys/class/hwmon/hwmon*/name` = `k10temp` (AMD) / `coretemp` (Intel) | ✓ | ✓ |
| GPU hwmon temp | `/sys/class/hwmon/hwmon*/name` = `amdgpu` (AMD) / `i915` or `xe` (Intel) | ✓ | ✓ |
| Perf profile | `/sys/devices/system/cpu/cpu0/cpufreq/energy_performance_preference` (EPP) | ✓ (AMD P-State EPP) | ✓ (Intel P-State EPP) |

Both `playos-platform-api/src/playos_power.c` and `playos-init/src/thermal.c`
classify hwmon `name` strings through vendor-agnostic matchers
(`playos__hwmon_name_is_gpu/cpu`) added in Sprint 13, so adding a vendor is a
name-list change, not a code-path fork.

## GPU description probe (libplayos)

`playos_system.c` reads `/sys/class/drm/card0/device/vendor` (with a `card1`
fallback) and maps PCI vendor IDs (`0x1002` AMD, `0x10de` NVIDIA, `0x8086`
Intel) to a human-readable GPU string. This is a **display string only** — GPU
*selection* lives in the compositor. Making this probe enumerate DRM devices is
deferred; the card-index fallback is acceptable on single-iGPU targets.

## Audio

- Both targets use ALSA via raylib's miniaudio backend (`MA_NO_PULSEAUDIO`).
- AMD Ally needs card routing (`asound.conf` → card 1). Intel HDA typically
  presents card 0; the shell retries audio init until the card registers.

## Mesa

- AMD: `gallium-drivers=radeonsi` (+ AMD Vulkan).
- Intel Gen 9.5 (HD 620): `gallium-drivers=iris` with `BR2_PACKAGE_MESA3D_LLVM=y`
  (iris depends on LLVM in Buildroot). Intel Vulkan (ANV) is deferred to a future
  sprint.
- EGL/ES/GBM stay identical across targets.
