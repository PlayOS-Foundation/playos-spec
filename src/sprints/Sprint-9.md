# Sprint 9 — Power, Battery, Thermal, and Suspend Foundations

**Goal:** Expose safe power, battery, and thermal information through the `playos-platform-api`. Implement AMD P-state integration for basic performance profiles. Establish thermal limits that prevent overheating. Lay the groundwork for suspend/resume (full suspend deferred to post-MVP).

**Primary Outcome:** Shell and overlay show live battery level and thermal state. A performance profile can be requested. Thermal throttling kicks in before dangerous temperatures. The ROG Ally does not overheat under sustained load.

**Status:** 🟢 Implemented and verified on Ally — Sprint 9 complete.

**Prerequisites:** Sprint 8 complete — full audio and lifecycle working. ✅ Satisfied (audio verified on Ally; gamepad wired into raylib).

---

## Why This Sprint Exists

Sprint 8 delivers audio but the device still has no power awareness — battery can die silently and the CPU can overheat. Sprint 9 adds live battery/thermal telemetry, safe P-state control, and the overlay power menu. It also lays the suspend/resume skeleton so that the lifecycle enum is complete for all subsequent work.

---

## Start Condition Checklist

- ✅ Sprint 8 complete: ALSA audio verified on the Ally (dmix + mute-via-switch fixes); audio diagnostics landed as `src/playos-init/src/audio_debug.c` (commits `512ae05`, `f80e44a`); gamepad wired into raylib `CORE.Input.Gamepad.*`.
- ✅ Graphics stack upgraded to GLES 3.0 (`playos-shell` `4ad17f6`) — the S9-T8 sustained-load/thermal validation now exercises the ES3 path (EGL negotiates ES3.2 on RDNA3).
- ⚠️ `CONFIG_X86_AMD_PSTATE=y` confirmed at `br2-external/board/ally/linux.config:190` (plus `ACPI_BATTERY`, `ACPI_AC`, `POWER_SUPPLY`, `THERMAL`), but `CONFIG_X86_AMD_PSTATE_EPP=y` is **missing** — it must be enabled for the `energy_performance_preference` sysfs node this sprint writes.
- `/sys/class/power_supply/BAT0/` exists on the Ally (verify during bringup — cannot check without hardware).
- ✅ `playos-overlay` exists with D-pad volume controls and already renders a hardcoded `Battery: 85%   Thermal: Normal` placeholder (main.c:386) to replace with live data.

---

## Decisions Locked for This Sprint

- **sysfs only:** battery, CPU/GPU temp, P-state reads via sysfs — no vendor ACPI/WMI calls this sprint
- **Poll interval:** thermal monitor runs at 1 Hz in `playos-init`; shell refreshes battery display every 30 seconds
- **P-state write authority:** only `playos-init` (root) writes to sysfs P-state paths; games request via IPC
- **Thermal thresholds:** values defined here are the locked defaults for MVP; overridable via `/data/config/thermal.json`
- **Suspend:** skeleton only — deliver lifecycle events, attempt `echo mem > /sys/power/state`; stability is post-MVP
- **TDP tuning:** defer vendor WMI/ACPI TDP control unless the device runs critically hot at `balance_performance`
- **Kernel P-state support:** enable `CONFIG_X86_AMD_PSTATE_EPP=y` in `br2-external/board/ally/linux.config` so `energy_performance_preference` exists (today only `CONFIG_X86_AMD_PSTATE=y` is set).
- **Event channel:** `ThermalStateChanged` and `PerfProfileChanged` reuse the existing Sprint-7 shell listener (`playos_trusted_register_shell` / `playos_trusted_shell_poll`); no new IPC transport.

---

## Scope

### In Scope

- `playos_power.h` API (battery, thermal, P-state)
- sysfs-backed implementation
- AMD P-state integration (EPP writes)
- Thermal monitoring loop in `playos-init`
- Shell status bar (battery %, thermal indicator)
- Overlay additions (temperatures, profile selector, power menu)
- Suspend/resume skeleton (lifecycle events + `/sys/power/state` attempt)

### Explicitly Out of Scope

- Full suspend/resume stability (post-MVP)
- Vendor TDP WMI/ACPI tuning (post-MVP unless safety-critical)
- AC adapter wattage tracking (post-MVP)
- Fan curve control (post-MVP)

---

## Required Repository Changes

| Repo | Required work |
|---|---|
| `playos-platform-api` | `playos_power.h`, sysfs-backed implementation |
| `playos-refdistro` | `playos-init` thermal monitor, enable `CONFIG_X86_AMD_PSTATE_EPP=y` in `board/ally/linux.config`, thermal.json default |
| `playos-shell` | Battery/thermal status bar |
| `playos-refdistro` (`src/playos-overlay`, committed in-tree) | Temperature display, profile selector, power menu |
| `playos-runtime` | Add `SetPerfProfile` request + `ThermalStateChanged`/`PerfProfileChanged` events via the existing shell listener (`Shutdown`/`Reboot` already exist from Sprint 5) |
| `playos-spec` | Thermal policy doc, power API spec |

---

## Expected Files and Directories

### `playos-platform-api`

```text
include/playos/playos_power.h
src/playos_power.c           # sysfs reader, IPC request for profile changes
```

### `playos-refdistro`

```text
src/playos-init/src/thermal.c                      # 1 Hz monitor loop, P-state writer, IPC notifier
br2-external/board/ally/linux.config               # add CONFIG_X86_AMD_PSTATE_EPP=y
br2-external/board/common/rootfs-overlay/data/config/thermal.json
```

---

## Agent Task Breakdown

### Task Status Grid

| Task ID | Task | Primary repo | Status | Notes / evidence |
|---|---|---|---|---|
| S9-T1 | Define and implement `playos_power.h` API | `playos-platform-api` | done | `playos_power.h` + full `playos_power.c`: sysfs battery/thermal/EPP reads + IPC client |
| S9-T2 | Implement sysfs-backed battery and temperature reads | `playos-platform-api` | done | `BAT0` capacity/status/time-to-*, `x86_pkg_temp`/`cpu_thermal`/`k10temp`, `amdgpu` hwmon; 1 s cache |
| S9-T3 | Implement AMD P-state EPP write and profile IPC | `playos-refdistro`, `playos-runtime` | done | `thermal.c` EPP writer; `SetPerfProfile` handler in `ipc_handler.c`; EPP enabled via `DEFAULT_MODE=3` |
| S9-T4 | Implement thermal monitoring loop in `playos-init` | `playos-refdistro` | done | `src/playos-init/src/thermal.c` 1 Hz tick, `thermal.json` thresholds, state machine + events |
| S9-T5 | Update shell status bar (battery, thermal indicator) | `playos-shell` | done | battery %/charging + colour-coded thermal + profile (main.c) |
| S9-T6 | Update overlay (temps, profile selector, power menu) | `playos-refdistro` (`src/playos-overlay`) | done | live power/thermal, D-pad profile selector, Sleep/Restart/Shutdown menu |
| S9-T7 | Implement suspend/resume skeleton | `playos-refdistro`, `playos-platform-api` | done | `PLAYOS_IPC_TYPE_SUSPEND` → `playos_suspend()`; lifecycle events handled |
| S9-T8 | Power and thermal validation on Ally | `playos-refdistro` | done | status bar/overlay live values, reboot/shutdown, sustained-load verified on Ally |

### S9-T1 — Define and implement `playos_power.h` API

> **Status:** `include/playos/playos_power.h` already exists and matches this spec exactly; `src/playos_power.c` is a stub returning -1. Remaining work is the implementation, not the API surface.

```c
typedef enum { PLAYOS_POWER_STATE_ON_BATTERY, PLAYOS_POWER_STATE_CHARGING,
               PLAYOS_POWER_STATE_CHARGED, PLAYOS_POWER_STATE_UNKNOWN } PlayOSPowerState;
typedef enum { PLAYOS_THERMAL_NORMAL, PLAYOS_THERMAL_WARM,
               PLAYOS_THERMAL_HOT, PLAYOS_THERMAL_CRITICAL } PlayOSThermalState;
typedef enum { PLAYOS_PERF_BALANCED, PLAYOS_PERF_POWER_SAVE,
               PLAYOS_PERF_PERFORMANCE } PlayOSPerfProfile;

typedef struct {
    PlayOSPowerState   power_state;
    int                battery_percent;   /* 0–100; -1 if unknown */
    int                minutes_remaining; /* -1 if unknown or charging */
    PlayOSThermalState thermal_state;
    int                cpu_temp_c;
    int                gpu_temp_c;
    PlayOSPerfProfile  active_profile;
} PlayOSPowerInfo;

int playos_power_get_info(PlayOSPowerInfo *info);
int playos_power_request_profile(PlayOSPerfProfile profile);
```

`playos_power_request_profile()` sends an IPC message to `playos-init`; it does NOT write sysfs directly.

**Done when:** API compiles, `playos_power_get_info()` returns a populated struct with real sysfs values on the Ally.

### S9-T2 — Implement sysfs-backed battery and temperature reads

| Data | sysfs path |
|---|---|
| Battery capacity | `/sys/class/power_supply/BAT0/capacity` |
| Charging state | `/sys/class/power_supply/BAT0/status` |
| Time to empty | `/sys/class/power_supply/BAT0/time_to_empty_now` (if available) |
| CPU temperature | `/sys/class/thermal/thermal_zone*/temp` (match `x86_pkg_temp` type) |
| GPU temperature | `/sys/class/drm/card*/device/hwmon/hwmon*/temp1_input` |
| AMD EPP current | `/sys/devices/system/cpu/cpu0/cpufreq/energy_performance_preference` |

- Cache reads for 1 second; avoid re-reading sysfs on every `playos_power_get_info()` call
- Return -1 for any unavailable value; never crash on missing sysfs node

**Done when:** `playos_power_get_info()` returns correct battery % and temperatures matching a second data source (e.g., `sensors` output).

### S9-T3 — Implement AMD P-state EPP write and profile IPC

> **Prereq:** `energy_performance_preference` only exists when `CONFIG_X86_AMD_PSTATE_EPP=y`. Add it to `br2-external/board/ally/linux.config` first; without it the driver exposes only `scaling_governor`.

Profile → EPP mapping:

| PlayOS profile | AMD EPP value |
|---|---|
| `PLAYOS_PERF_BALANCED` | `balance_performance` |
| `PLAYOS_PERF_POWER_SAVE` | `power` |
| `PLAYOS_PERF_PERFORMANCE` | `performance` |

- `playos-init` receives `SetPerfProfile { profile }` IPC
- Validates the request against current thermal state (reject `PERFORMANCE` if `HOT` or `CRITICAL`)
- Writes EPP string to all online CPUs: `/sys/devices/system/cpu/cpuN/cpufreq/energy_performance_preference`
- Emits `PerfProfileChanged { profile }` event over the Sprint-7 shell listener

**Done when:** `SetPerfProfile { PLAYOS_PERF_PERFORMANCE }` is accepted when thermal state is NORMAL; rejected when HOT.

### S9-T4 — Implement thermal monitoring loop in `playos-init`

- 1 Hz poll loop in `src/playos-init/src/thermal.c`
- Read CPU and GPU temperatures via sysfs (reuse the reader from S9-T2)
- Compute `PlayOSThermalState` using thresholds from `/data/config/thermal.json` (or built-in defaults)

Default thresholds:
- NORMAL: < 75°C
- WARM: 75–85°C
- HOT: 85–95°C
- CRITICAL: ≥ 95°C

Actions:
- WARM: log; emit `ThermalStateChanged` event
- HOT: apply `PLAYOS_PERF_BALANCED`; emit event; log
- CRITICAL: apply `PLAYOS_PERF_POWER_SAVE`; emit event with warning flag; if unresolved in 10 s, call graceful shutdown

**Done when:** running a stress test on the Ally causes the thermal state to progress to WARM and the log shows the state change and P-state adjustment.

### S9-T5 — Update shell status bar

Add to the shell's bottom status bar:
- Battery percentage + charging indicator (⚡ when charging)
- Thermal state color indicator (green = NORMAL, yellow = WARM, red = HOT/CRITICAL)
- Active performance profile indicator
- Refresh every 30 seconds (or immediately on a `ThermalStateChanged` / `PerfProfileChanged` event from the Sprint-7 shell listener)

**Done when:** plugging and unplugging the charger updates the charging indicator within 30 s; running a stress test changes the thermal indicator to yellow.

### S9-T6 — Update overlay (temperatures, profile selector, power menu)

> **Status:** overlay already renders a hardcoded `Battery: 85%   Thermal: Normal` line (main.c:386) and D-pad volume controls; replace the placeholder with live `playos_power_get_info()` data. The profile selector's D-pad + A confirm is now available via the Sprint-8 gamepad wiring.

Add to `playos-overlay`:
- Battery percentage and estimated time remaining
- CPU and GPU temperature (live, refreshed every 5 s while overlay is shown)
- Performance profile selector: D-pad to cycle, A to confirm → sends `SetPerfProfile` IPC
- Power menu: "Sleep" (disabled placeholder), "Restart", "Shutdown"
- Restart: sends `Reboot` IPC to `playos-init`
- Shutdown: sends `Shutdown` IPC to `playos-init`

**Done when:** overlay shows live temperatures; profile selector changes the active profile; Restart reboots the device cleanly.

### S9-T7 — Implement suspend/resume skeleton

- `PLAYOS_LIFECYCLE_SUSPEND` (0x02) and `PLAYOS_LIFECYCLE_RESUME` (0x03) are already defined in `playos-init/ipc/ipc.h`; this task only ensures every lifecycle consumer handles them without crashing
- On lid close event or suspend button: deliver `PLAYOS_LIFECYCLE_SUSPEND` to any running game; attempt `echo mem > /sys/power/state`
- If the write fails: log the error; continue running
- On resume (if successful): deliver `PLAYOS_LIFECYCLE_RESUME`
- The "Sleep" power menu item sends the suspend trigger

**Done when:** `PLAYOS_LIFECYCLE_SUSPEND` and `PLAYOS_LIFECYCLE_RESUME` are delivered without crashing any lifecycle consumer. Actual device sleep is best-effort this sprint.

### S9-T8 — Power and thermal validation on Ally

- Battery display: run on battery 10 min; verify percentage decreases and display updates
- Charging indicator: plug/unplug charger; verify indicator updates
- Thermal progression: run CPU/GPU stress tool; verify NORMAL → WARM state change and log
- P-state change: verify EPP file content before and after `SetPerfProfile` IPC
- Shutdown and restart via overlay: verify clean filesystem state after restart
- Suspend skeleton: `PLAYOS_LIFECYCLE_SUSPEND` delivered; device may or may not sleep; no crash

**Done when:** all validation cases produce expected log entries and behavior.

---

## Implementation Guidance

### sysfs thermal zone selection

There may be multiple thermal zones. Select the zone whose `type` file contains `x86_pkg_temp` for CPU. For GPU, look for `amdgpu` hwmon entry. If neither is found, return -1 and log once (not every poll).

### `thermal.json` format

```json
{
  "thresholds": {
    "warm_c": 75,
    "hot_c": 85,
    "critical_c": 95
  }
}
```

If the file is missing or malformed, use the built-in defaults and log a warning.

---

## Verification and Evidence

| Evidence | How it is produced |
|---|---|
| Battery accuracy | `playos_power_get_info()` output vs. `cat /sys/class/power_supply/BAT0/capacity` |
| Temperature accuracy | `playos_power_get_info()` output vs. `sensors` |
| Thermal state progression | `playos-init` log during stress test |
| P-state change | `/sys/devices/system/cpu/cpu0/cpufreq/energy_performance_preference` before/after |
| Profile rejection | Log showing `SetPerfProfile PERFORMANCE` rejected when HOT |
| Restart clean | `journalctl` showing clean shutdown sequence |

---

## Acceptance Criteria

- [ ] Shell status bar shows live battery percentage; updates within 30 s of charging state change
- [ ] CPU and GPU temperatures visible in overlay; accurate within ±2°C of `sensors`
- [ ] Running a stress test progresses thermal state from NORMAL to WARM; log shows state change
- [ ] At HOT state: system switches to BALANCED P-state; overlay shows warning
- [ ] `SetPerfProfile PERFORMANCE` honored when thermal state is NORMAL
- [ ] `SetPerfProfile PERFORMANCE` rejected when thermal state is HOT
- [ ] Shutdown from overlay: system shuts down cleanly (filesystems synced)
- [ ] Restart from overlay: system reboots cleanly
- [ ] `PLAYOS_LIFECYCLE_SUSPEND` and `PLAYOS_LIFECYCLE_RESUME` delivered without crashing any consumer
- [ ] Sample games run sustained load without GPU hang or kernel panic
- [ ] CI passes

---

## Handoff to Sprint 10

Sprint 10 may assume:

- `playos_power.h` API is stable and returns real data
- Shutdown and Reboot IPC commands are implemented and tested
- Thermal thresholds are configurable via `/data/config/thermal.json`
- Suspend skeleton exists; the event lifecycle is complete
- The overlay is a stable client that can receive further extension (update progress UI, Sprint 11)

---

## Exit Gate

Battery, thermal, and power status are live in the shell and overlay. Thermal throttling prevents dangerous temperatures. Performance profiles can be requested by games and set via the overlay.

*Previous: [Sprint 8](Sprint-8.md) | Next: [Sprint 10](Sprint-10.md)*
