# Game Performance

PlayOS games are expected to hold a stable frame rate at the display refresh
rate (60 Hz or 120 Hz on supported panels).

## Baseline expectations (Sprint 14 T7)

- Boot-to-shell: **< 10 s** on device (target)
- Shell idle: **< 10% CPU** on one core
- Game foreground: maintain refresh rate at native resolution
- Background: **near-zero CPU** (lifecycle-paused)
- Suspend: save + return **within 500 ms**
- Thermals: stay under `PLAYOS_THERMAL_HOT` during normal play

## How to measure

- Shell/status bar shows FPS + power/thermal info.
- `playos_power_get_info()` for CPU/GPU temps and active perf profile.
- Device logs: `/data/log/` per-game session logs with frame timings.
- Dev images: SSH + `top`, `powertop` on the device.

## Performance profiles

Games may request `PLAYOS_PERF_BALANCED`, `PLAYOS_PERF_POWER_SAVE`, or
`PLAYOS_PERF_PERFORMANCE` via `playos_power_request_profile()`. The system may
deny or override based on thermal state.

## Rules

- Do not busy-wait in BACKGROUND.
- Poll input/lifecycle at frame rate; avoid spin loops.
- Prefer `playos_storage_atomic_write` over frequent synchronous fsyncs.
