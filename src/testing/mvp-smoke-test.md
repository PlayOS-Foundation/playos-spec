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
| 2 | UEFI-bootable EFI artifact | `ls /EFI/BOOT/BOOTX64.EFI` on live ESP | ☐ | |
| 3 | `playos-init` is PID 1 | SSH: `awk '{print $4}' /proc/1/comm` → `playos-init` | ☐ | |
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
| 19 | Recovery usable without accelerated graphics | Hold Vol-Down 5s at boot → recovery menu (SimpleDRM follow-up) | ☐ | |

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
