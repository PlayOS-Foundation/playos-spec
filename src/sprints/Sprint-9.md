# Sprint 9 — Power, Battery, Thermal, and Suspend Foundations

**Goal:** Expose safe power, battery, and thermal information through the `playos-platform-api`. Implement AMD P-state integration for basic performance profiles. Establish thermal limits that prevent overheating. Lay the groundwork for suspend/resume (full suspend deferred to post-MVP).

**Primary Outcome:** Shell and overlay show live battery level and thermal state. A performance profile can be requested. Thermal throttling kicks in before dangerous temperatures. The ROG Ally does not overheat under sustained load.

**Prerequisites:** Sprint 8 complete — full audio and lifecycle working.

---

## Why This Sprint Exists

Sprint 8 delivers audio but the device still has no power awareness — battery can die silently and the CPU can overheat. Sprint 9 adds live battery/thermal telemetry, safe P-state control, and the overlay power menu. It also lays the suspend/resume skeleton so that the lifecycle enum is complete for all subsequent work.

---

## Start Condition Checklist

- Sprint 8 complete: audio and full lifecycle working on the Ally.
- Kernel config includes `CONFIG_X86_AMD_PSTATE` (confirm in Sprint 1 defconfig).
- `/sys/class/power_supply/BAT0/` exists on the Ally (verify during Sprint 9 bringup).
- `playos-overlay` exists with volume controls; this sprint extends it.

---

## Decisions Locked for This Sprint

- **sysfs only:** battery, CPU/GPU temp, P-state reads via sysfs — no vendor ACPI/WMI calls this sprint
- **Poll interval:** thermal monitor runs at 1 Hz in `playos-init`; shell refreshes battery display every 30 seconds
- **P-state write authority:** only `playos-init` (root) writes to sysfs P-state paths; games request via IPC
- **Thermal thresholds:** values defined here are the locked defaults for MVP; overridable via `/data/config/thermal.json`
- **Suspend:** skeleton only — deliver lifecycle events, attempt `echo mem > /sys/power/state`; stability is post-MVP
- **TDP tuning:** defer vendor WMI/ACPI TDP control unless the device runs critically hot at `balance_performance`

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
| `playos-refdistro` | `playos-init` thermal monitor, AMD P-state in kernel defconfig, thermal.json default |
| `playos-shell` | Battery/thermal status bar |
| `playos-overlay` | Temperature display, profile selector, power menu |
| `playos-runtime` | `SetPerfProfile`, `ThermalStateChanged`, `Shutdown`, `Reboot` IPC commands |
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
src/playos-init/src/thermal.c    # 1 Hz monitor loop, P-state writer, IPC notifier
br2-external/board/common/rootfs-overlay/data/config/thermal.json
```

---

## Agent Task Breakdown

### Task Status Grid

| Task ID | Task | Primary repo | Status | Notes / evidence |
|---|---|---|---|---|
| S9-T1 | Define and implement `playos_power.h` API | `playos-platform-api` | not started | |
| S9-T2 | Implement sysfs-backed battery and temperature reads | `playos-platform-api` | not started | |
| S9-T3 | Implement AMD P-state EPP write and profile IPC | `playos-refdistro`, `playos-runtime` | not started | |
| S9-T4 | Implement thermal monitoring loop in `playos-init` | `playos-refdistro` | not started | |
| S9-T5 | Update shell status bar (battery, thermal indicator) | `playos-shell` | not started | |
| S9-T6 | Update overlay (temps, profile selector, power menu) | `playos-overlay` | not started | |
| S9-T7 | Implement suspend/resume skeleton | `playos-refdistro`, `playos-platform-api` | not started | |
| S9-T8 | Power and thermal validation on Ally | `playos-refdistro` | not started | |

### S9-T1 — Define and implement `playos_power.h` API

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

Profile → EPP mapping:

| PlayOS profile | AMD EPP value |
|---|---|
| `PLAYOS_PERF_BALANCED` | `balance_performance` |
| `PLAYOS_PERF_POWER_SAVE` | `power` |
| `PLAYOS_PERF_PERFORMANCE` | `performance` |

- `playos-init` receives `SetPerfProfile { profile }` IPC
- Validates the request against current thermal state (reject `PERFORMANCE` if `HOT` or `CRITICAL`)
- Writes EPP string to all online CPUs: `/sys/devices/system/cpu/cpuN/cpufreq/energy_performance_preference`
- Emits `PerfProfileChanged { profile }` event to subscribers

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
- Refresh every 30 seconds (or immediately on `ThermalStateChanged` / `PerfProfileChanged` IPC event)

**Done when:** plugging and unplugging the charger updates the charging indicator within 30 s; running a stress test changes the thermal indicator to yellow.

### S9-T6 — Update overlay (temperatures, profile selector, power menu)

Add to `playos-overlay`:
- Battery percentage and estimated time remaining
- CPU and GPU temperature (live, refreshed every 5 s while overlay is shown)
- Performance profile selector: D-pad to cycle, A to confirm → sends `SetPerfProfile` IPC
- Power menu: "Sleep" (disabled placeholder), "Restart", "Shutdown"
- Restart: sends `Reboot` IPC to `playos-init`
- Shutdown: sends `Shutdown` IPC to `playos-init`

**Done when:** overlay shows live temperatures; profile selector changes the active profile; Restart reboots the device cleanly.

### S9-T7 — Implement suspend/resume skeleton

- Add `PLAYOS_LIFECYCLE_SUSPEND` and `PLAYOS_LIFECYCLE_RESUME` to the enum (already defined — ensure they are handled without crash in all lifecycle consumers)
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

Define `include/playos/playos_power.h`:

```c
typedef enum {
    PLAYOS_POWER_STATE_ON_BATTERY,
    PLAYOS_POWER_STATE_CHARGING,
    PLAYOS_POWER_STATE_CHARGED,
    PLAYOS_POWER_STATE_UNKNOWN
} PlayOSPowerState;

typedef enum {
    PLAYOS_THERMAL_NORMAL,
    PLAYOS_THERMAL_WARM,
    PLAYOS_THERMAL_HOT,
    PLAYOS_THERMAL_CRITICAL
} PlayOSThermalState;

typedef enum {
    PLAYOS_PERF_BALANCED,       /* Default — system-managed */
    PLAYOS_PERF_POWER_SAVE,     /* Low TDP — extend battery */
    PLAYOS_PERF_PERFORMANCE,    /* High TDP — better GPU/CPU, shorter battery */
} PlayOSPerfProfile;

typedef struct {
    PlayOSPowerState   power_state;
    int                battery_percent;   /* 0–100; -1 if unknown */
    int                minutes_remaining; /* -1 if unknown or charging */
    PlayOSThermalState thermal_state;
    int                cpu_temp_c;        /* -1 if unknown */
    int                gpu_temp_c;        /* -1 if unknown */
    PlayOSPerfProfile  active_profile;
} PlayOSPowerInfo;

int playos_power_get_info(PlayOSPowerInfo *info);

/* Request a performance profile. PlayOS may reject or delay the change.
   Returns 0 if the request was accepted, -1 if denied. */
int playos_power_request_profile(PlayOSPerfProfile profile);
```

**Implementation — read from sysfs:**

| Data | sysfs path |
|---|---|
| Battery capacity | `/sys/class/power_supply/BAT0/capacity` |
| Charging state | `/sys/class/power_supply/BAT0/status` |
| Time to empty | `/sys/class/power_supply/BAT0/time_to_empty_now` (if available) |
| CPU temperature | `/sys/class/thermal/thermal_zone*/temp` (find "x86_pkg_temp") |
| GPU temperature | `/sys/class/drm/card*/device/hwmon/hwmon*/temp1_input` |
| AMD P-state | `/sys/devices/system/cpu/cpu0/cpufreq/energy_performance_preference` |

### AMD P-state Integration

The ROG Ally uses AMD CPU with P-state support (`CONFIG_X86_AMD_PSTATE`).

**Profile mapping to AMD P-state hints:**

| PlayOS profile | AMD EPP value |
|---|---|
| `PLAYOS_PERF_BALANCED` | `balance_performance` |
| `PLAYOS_PERF_POWER_SAVE` | `power` |
| `PLAYOS_PERF_PERFORMANCE` | `performance` |

Write the EPP string to all online CPUs:
```
/sys/devices/system/cpu/cpuN/cpufreq/energy_performance_preference
```

Only `playos-init` (root) may write to these sysfs paths. Games request a profile through `playos_power_request_profile()`, which goes via IPC to `playos-init` for validation and application.

**TDP control (if supported by the Ally firmware via ACPI/WMI):**
- Defer vendor-specific TDP tuning to post-MVP unless it is required for safe operation
- If the device runs dangerously hot at max P-state, apply a safe TDP cap by default

### Thermal Management

**Thermal zones to monitor:**
- CPU package temperature
- GPU temperature
- SoC/APU composite (if available)

**Thermal state thresholds (tunable via `/data/config/thermal.json`):**

| State | CPU/GPU temp |
|---|---|
| NORMAL | < 75°C |
| WARM | 75–85°C |
| HOT | 85–95°C |
| CRITICAL | ≥ 95°C |

**Thermal actions:**

| State | Action |
|---|---|
| WARM | Log; show thermal indicator in overlay status bar |
| HOT | Switch to `PLAYOS_PERF_BALANCED`; log; notify shell |
| CRITICAL | Force `PLAYOS_PERF_POWER_SAVE`; display warning in overlay; if unresolved in 10s, graceful shutdown |

`playos-init` owns the thermal monitoring loop (1 Hz poll). It applies P-state changes. The compositor/shell are notified via IPC events.

### Shell and Overlay Updates

**Shell status bar (bottom):**
- Battery percentage + charging indicator (⚡)
- Thermal state indicator (color: green/yellow/red)
- Active performance profile indicator

**Overlay additions:**
- Battery percentage, estimated time remaining
- Current GPU and CPU temperatures
- Performance profile selector (D-pad to cycle, A to confirm)
- Power menu: "Sleep" (disabled — placeholder), "Restart", "Shutdown"

**Shutdown/Reboot from overlay:** Call `playos-init`'s `Shutdown` or `Reboot` IPC command.

### Suspend/Resume Foundations (Skeleton Only)

Full suspend/resume is a post-MVP feature, but lay the foundations:
- Add `PLAYOS_LIFECYCLE_SUSPEND` and `PLAYOS_LIFECYCLE_RESUME` to the lifecycle enum (already defined, now ensure they are handled gracefully)
- On lid close or suspend key press: deliver `PLAYOS_LIFECYCLE_SUSPEND` to the game, then attempt a system suspend via `/sys/power/state = mem`
- If suspend fails: log the error; remain running
- On resume: deliver `PLAYOS_LIFECYCLE_RESUME`

For this sprint: test that lifecycle events are delivered correctly. Actual suspend/resume stability is post-MVP.

---

## Acceptance Criteria

- [ ] Shell status bar shows live battery percentage; updates every 30 seconds
- [ ] Charging state is reflected correctly (shows ⚡ when plugged in)
- [ ] CPU and GPU temperatures visible in overlay
- [ ] Thermal state changes from NORMAL → WARM at 75°C (simulate by running a stress test)
- [ ] At HOT state: system switches to balanced P-state; overlay shows warning
- [ ] `playos_power_request_profile(PLAYOS_PERF_PERFORMANCE)` is honored when thermal state is NORMAL
- [ ] `playos_power_request_profile(PLAYOS_PERF_PERFORMANCE)` is denied or overridden when thermal state is HOT
- [ ] Shutdown from overlay: system shuts down cleanly (filesystems synced)
- [ ] Restart from overlay: system reboots cleanly
- [ ] `PLAYOS_LIFECYCLE_SUSPEND` and `PLAYOS_LIFECYCLE_RESUME` are delivered without crashing
- [ ] Running `com.playos.sample-triangle` under sustained load does not cause GPU hang or kernel panic
- [ ] CI passes

---

## Repositories Primarily Involved

| Repo | Work |
|---|---|
| `playos-platform-api` | `playos_power.h` API, sysfs-backed implementation |
| `playos-refdistro` | Thermal config, AMD P-state in kernel config, `playos-init` thermal monitor |
| `playos-shell` | Battery/thermal status bar |
| `playos-overlay` | Temperature, profile selector, power menu |
| `playos-runtime` | `SetPerfProfile`, `ThermalStateChanged` IPC events |
| `playos-spec` | Thermal policy document, power API spec |

---

## Testing Approach

- Physical ROG Ally required
- Battery test: run sample game for 10 minutes; verify percentage decreases on battery
- Thermal test: run a CPU/GPU stress tool; verify thermal state progression and P-state downgrade
- Shutdown/restart: verify via overlay controls; verify filesystem integrity after restart
- Suspend skeleton: deliver suspend event; verify game pauses; resume; verify game continues

---

## Exit Gate

Battery, thermal, and power status are live in the shell and overlay. Thermal throttling prevents dangerous temperatures. Performance profiles can be requested by games and set via the overlay.

*Previous: [Sprint 8](Sprint-8.md) | Next: [Sprint 10](Sprint-10.md)*
