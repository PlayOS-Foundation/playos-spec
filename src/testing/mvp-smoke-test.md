# MVP Smoke Test (Sprint 14 T5)

The MVP is complete only when all 19 roadmap criteria pass on physical ROG
Ally hardware. This document is the repeatable smoke-test checklist.

Source: [`sprints/roadmap.md`](../sprints/roadmap.md)

## Prerequisites

- Latest **ally dev USB image** (SSH enabled) flashed to USB
- Physical ROG Ally (charger + controller input via built-in gamepad)
- SSH access: `ssh root@<device-ip>`

## Checklist

| # | Criterion | Verify by | PASS | Notes |
|---:|---|---|:---:|---|
| 1 | Boots directly from UEFI into PlayOS | Power on → shell visible, no desktop | ☐ | |
| 2 | UEFI-bootable EFI artifact | `ls /EFI/EFI/BOOT/BOOTX64.EFI` (ESP mounted at `/EFI`; on the live USB it is `/EFI/BOOT/BOOTX64.EFI`) | ☐ | |
| 3 | `playos-init` is PID 1 | `readlink /proc/1/exe` → `/init` and `strings /init \| grep -m1 playos-init`. (`/proc/1/comm` is just `init` — the binary is installed as `/init`; `/sbin/init` is BusyBox) | ☐ | |
| 4 | Compositor owns DRM/KMS + Wayland | SSH: `pgrep playos-compositor`, `/run/playos/playos-0` socket | ☐ | |
| 5 | Shell persistent controller-first UI | Home screen stays alive; background/return works | ☐ | |
| 6 | wlroots + AMDGPU DRM/GBM/EGL/Mesa | `grep -E "AMDGPU|Radeon" /data/log/init.log`; compositor log | ☐ | |
| 7 | Shell renders via Raylib PlayOS backend | Shell visible, sample game renders | ☐ | |
| 8 | Shell + game use public `libplayos` ABI | `ldd /usr/bin/playos-shell` → `libplayos.so.0` | ☐ | |
| 9 | Trusted transport stays internal | No public socket for game control; only `/run/playos/control.sock` + compositor.sock | ☐ | |
| 10 | Shell requests launch; init spawns/supervises | Launch sample game; SSH `pgrep -a playos-game` | ☐ | |
| 11 | Compositor waits for first valid frame | Quick-launch sample; no flash/black flicker | ☐ | |
| 12 | Hardware-accelerated render + controller input | Sample game: movement/buttons respond | ☐ | |
| 13 | System button backgrounds/pauses game | Press System → shell returns; game pauses | ☐ | |
| 14 | Resume returns to same game without restart | Press System again → same game state | ☐ | |
| 15 | Game audio through ALSA | Sample game plays audio; `aplay -l` shows device | ☐ | |
| 16 | Clean exit + crash return to shell | Exit sample; run crash sample → shell recovers | ☐ | |
| 17 | Games/saves on separate ext4 | `mount | grep /data` → ext4; saves persist reboot | ☐ | |
| 18 | System image immutable | `/` mounted squashfs ro: `mount | grep " / "` | ☐ | |
| 19 | Recovery usable without accelerated graphics | Boot with a recovery hold (START+SELECT 2 s, or Vol Up/Down 5 s) → recovery menu | ☑ | 2026-09-13: **met** via the GL-free recovery client (`playos-recovery`, wl_shm + software text), which init starts when the GL shell cannot run in recovery; compositor-side software rendering over SimplEDRM verified in QEMU with a screenshot, and the client path verified with the `playos.noshell` hook. Evidence `playos-refdistro/docs/evidence/f3-recovery-client-no-gl-2026-09-13.png`, report `docs/f3-recovery-software-rendering-2026-09-13.md`. Not covered: a machine with no DRM device at all (no compositor) would need a kernel-console UI |

## Automated evidence

Run on the device (dev image):

```sh
sh scripts/mvp-smoke.sh > mvp-evidence.md
```

The script captures process checks, sockets, DRM/ALSA devices, mounts, and
boot-log markers for the automatable criteria.

## Evidence record

```markdown
## MVP smoke test — <date> — ROG Ally — image <commit>

All 19 criteria PASS: ☐ / 19

Attach: mvp-evidence.md, shell screenshot, audio log.
```

### 2026-09-12 — dev image — 18 / 19

- Report: `playos-refdistro/docs/mvp-smoke-report-2026-09-12.md`
- Raw collector output: `playos-refdistro/docs/evidence/mvp-evidence-2026-09-12.md`
- Image: shell `0a76f93f…`, compositor `0837553f…`, overlay `f0394108…`,
  libplayos `46000d67…` (refdistro `f0bd607`)
- **Criterion 19 PASSES (2026-09-13)**: the recovery UI no longer needs GL at
  all. `playos-recovery` (wl_shm + software text rasteriser) takes over when the
  GL shell cannot run, and the compositor itself can render in software over
  SimplEDRM with no GPU driver. Verified in QEMU (client menu on screen) with the
  `playos.noshell` hook reproducing the shell-failure case — see the F3 report.
  Every other criterion passed, with controller input, overlay background/resume,
  screenshots, clean exit and crash recovery all exercised on hardware.
