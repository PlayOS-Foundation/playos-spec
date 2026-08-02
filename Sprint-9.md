# Sprint 9 — Power, Battery, Thermal, and Suspend Foundations

**Goal:** Expose safe power, battery, and thermal information through the `playos-platform-api`. Implement AMD P-state integration for basic performance profiles. Establish thermal limits that prevent overheating. Lay the groundwork for suspend/resume (full suspend deferred to post-MVP).

**Primary Outcome:** Shell and overlay show live battery level and thermal state. A performance profile can be requested. Thermal throttling kicks in before dangerous temperatures. The ROG Ally does not overheat under sustained load.

**Prerequisites:** Sprint 8 complete — full audio and lifecycle working.

---

## Key Deliverables

### `playos-platform-api` — Power API

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
