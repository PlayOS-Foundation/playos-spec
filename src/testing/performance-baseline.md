# Performance Baseline (Sprint 14 T7)

The performance baseline is the acceptance gate for "feels like a console"
performance on the ROG Ally (and other supported targets).

## Baseline targets

| Metric | Target | How measured |
|---|---|---|
| Boot to shell | **< 10 s** on device | stopwatch from power-on to shell visible (dev USB image) |
| Shell idle CPU | **< 10%** of one core | `top`/`pidstat` on dev image, shell idle at home |
| Game foreground | stable at display refresh (60/120 Hz) | shell FPS overlay / sample game |
| Game background | **near-zero CPU** | lifecycle-paused game via `top` |
| Suspend save+return | **within 500 ms** | dev image suspend test |
| Thermal state | stays under `PLAYOS_THERMAL_HOT` (85°C) in normal play | `playos_power_get_info()` / hwmon |

## Measurement script

A repeatable collector ships in `playos-refdistro/scripts/perf-baseline.sh`.

On the device:

```sh
sh /root/perf-baseline.sh > perf-report.md
```

Or over SSH on a dev image:

```sh
ssh root@<device-ip> 'sh -s' < scripts/perf-baseline.sh > perf-report.md
```

The report captures: device model, uptime, load average, memory, CPU model,
thermal zones, power supply state, and shell FPS from device logs.

## Manual checklist

1. Boot the latest dev USB image on the target.
2. Start the stopwatch at power-on; record when the home screen is interactive.
3. Leave the shell idle for 60 s; sample CPU with `top` (dev image SSH).
4. Launch the sample game; verify stable frame rate at native resolution.
5. Press the system button to background the game; verify CPU drops to ~0.
6. Trigger suspend/resume; verify return-to-game < 500 ms.
7. Check thermal state during a 10-minute play session.

## Evidence record

Append one section per test run to the sprint evidence:

```markdown
## Performance baseline — <date> — <device> — <image commit>

| Metric | Target | Result | PASS? |
|---|---|---|---|
| Boot to shell | < 10 s | 8.2 s | ✅ |
| Shell idle CPU | < 10% | 6% | ✅ |
| Game foreground | 60 Hz stable | 60 fps | ✅ |
| Game background CPU | ~0% | 1% | ✅ |
| Suspend return | < 500 ms | 320 ms | ✅ |
| Thermal | < 85°C | 71°C | ✅ |

Attach: `perf-report.md`
```
