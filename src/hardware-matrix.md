# Supported Hardware Matrix

> Last updated: 2026-08-24. Sprint 13 in progress.

| Target | Class | GPU | Audio | Power | Input | Status |
|---|---|---|---|---|---|---|
| **ASUS ROG Ally (2023)** | handheld (primary) | AMD Ryzen Z1 Extreme iGPU — `amdgpu`, Mesa `radeonsi` | ALC294 + CS35L41 (card 1; `asound.conf` routes to card 1) | AMD P-State EPP, k10temp hwmon, BAT0 | built-in gamepad + ASUS EC (root-only) | **Supported — Sprints 0–12 validated on-device** |
| **ASUS ZenBook UX530 (UX530UQ/UX530UX)** | laptop (Sprint 13) | Intel HD Graphics 620 (Gen 9.5) — `i915`, Mesa `iris`; NVIDIA 940MX/GTX 950 present but unselected (scores +1 as "other") | Intel HDA + Realtek codec | Intel P-State EPP + RAPL, coretemp hwmon, BAT0 | USB gamepad (no built-in) | **In progress — config authored, on-device validation pending** |
| **QEMU x86_64 + OVMF** | dev / CI | virtio-gpu / softpipe | none (smoke) | none | none | **Supported — CI build + boot smoke** |

## Kernel driver map

| Function | AMD (ROG Ally) | Intel (ZenBook UX530) |
|---|---|---|
| DRM driver | `CONFIG_DRM_AMDGPU` | `CONFIG_DRM_I915` (XE is Gen 12+ only) |
| Mesa Gallium | `BR2_PACKAGE_MESA3D_GALLIUM_DRIVER_RADEONSI` | `BR2_PACKAGE_MESA3D_GALLIUM_DRIVER_IRIS` (+ `BR2_PACKAGE_MESA3D_LLVM=y`) |
| Audio | ACP/`snd-acp` (AMD) | `CONFIG_SND_HDA_INTEL` + `CONFIG_SND_HDA_CODEC_REALTEK` |
| CPU freq / perf | `CONFIG_X86_AMD_PSTATE` | `CONFIG_X86_INTEL_PSTATE` |
| Power capping | — | `CONFIG_INTEL_RAPL` |
| CPU hwmon temp | `k10temp` | `coretemp` |
| GPU hwmon temp | `amdgpu` | `i915` |

Explicitly disabled on each vendor config: the other vendor's DRM/pstate driver, plus
`CONFIG_DRM_NOUVEAU`/`BR2_PACKAGE_NVIDIA_DRIVER` on the Intel image (the ZenBook's
NVIDIA dGPU must not bind a driver).

## Validation evidence

- ROG Ally: Sprints 0–12 acceptance runs (see `sprints/Sprint-*.md`).
- ZenBook UX530: `sprints/Sprint-13.md` S13-T1..T8 — host/QEMU parts landed; on-device matrix pending hardware.
- QEMU: `make qemu-build` + `scripts/qemu-boot-check.sh` in CI.
