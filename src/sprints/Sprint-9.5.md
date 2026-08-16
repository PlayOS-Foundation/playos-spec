# Sprint 9.5 — Display Brightness Control

**Goal:** Add a working display-brightness primitive to the `playos-platform-api` and surface it as an interactive control on the Settings → Display tab, so the ROG Ally's panel backlight can be read and adjusted from the shell.

**Primary Outcome:** The Settings → Display tab shows a live Brightness gauge (0–100%). D-pad up/down adjusts the backlight and the change is written through `/sys/class/backlight/` immediately, with the value reflected on the next read.

**Status:** 🟢 Implemented — platform-api + shell changes landed; on-device brightness verification pending

**Prerequisites:** Sprint 9 complete — power/battery/thermal telemetry and profile IPC are in place, and the Settings screen already renders tabs with a read-only info layout. ✅ Satisfied (`playos_power.c` ships the sysfs-read + 1-second-cache + IPC patterns this sprint reuses).

---

## Why This Sprint Exists

The device profiles already advertise the capability:

- `playos-reference-devices/rog-ally/device-profile.toml:34` — `"display.brightness" = true`
- `playos-reference-devices/asus-ultrabook/device-profile.toml:24` — `"display.brightness" = true`

But nothing in the code reads or writes a backlight. Investigation confirmed:

1. `playos-platform-api/include/playos/playos_display.h` already exists (and is already pulled in by `playos.h`), but `src/playos_display.c` is a stub — `playos_display_get_info()` and `playos_display_set_vsync()` both just `return -1`.
2. No sysfs backlight access exists anywhere in `playos-platform-api`, `playos-init`, `playos-runtime`, or `playos-shell`.
3. The Settings → Display tab is read-only (`playos-shell/src/screen_settings.c:616-632` shows only Resolution / DPI Scale / GPU), and its update path falls into the generic "read-only tab: scroll" branch (`screen_settings.c:218-226`) — there is no per-row interactive control outside the System tab.
4. The spec still marks brightness as a stub (`playos-spec/src/playos-shell-spec.md:47`), while the architecture assigns a "volume/brightness HUD" to the overlay (`playos-spec/src/architecture.md:177`) — that HUD is a separate follow-up, not this sprint.

This is a **small, additive sprint**: no new public header, no ABI break, and the shell already has all the rendering helpers and cached-state patterns needed for a gauge row.

---

## Start Condition Checklist

- [x] `playos_display.h` exists with a `PlayOSDisplayInfo` struct and is included by the master `playos.h` include.
- [x] `playos_power.c` demonstrates the sysfs-read + 1-second monotonic cache + `/sys/class/*` enumeration patterns to copy.
- [x] The Settings screen has `draw_info_line()` and `draw_trigger_gauge()` helpers to reuse for the brightness row.
- [x] Shell state struct (`playos-shell/include/shell.h`) has cached power info (`power_info` / `power_info_valid`) and settings tab/cursor fields to extend.
- [ ] Device-side check: the Ally exposes a backlight node under `/sys/class/backlight/` (expected `amdgpu_bl0`) and the shell's uid can write its `brightness` file. *(Cannot be confirmed from the build tree — verify on hardware before the write-path decision.)*

---

## Decisions Locked for This Sprint

- **API home:** extend the existing `playos_display.h` / `playos_display.c`. Do **not** create a new header and do **not** overload `playos_power.h`.
- **Interface (additive):**
  - `int playos_display_get_brightness(int *percent)` — returns 0..100, or -1 when unsupported.
  - `int playos_display_set_brightness(int percent)` — clamps 0..100 and writes the scaled raw value.
- **sysfs only:** no vendor WMI/ACPI backlight calls this sprint.
- **Node preference:** `amdgpu_bl0` → `acpi_video0` → `intel_backlight` → first other non-`acpi_` entry under `/sys/class/backlight/`. Skip entries with `max_brightness == 0`.
- **Percent mapping:** `percent = round(brightness * 100 / max_brightness)`; on set, `raw = clamp(round(percent * max_brightness / 100), 0, max_brightness)`. `max_brightness` is read once and cached.
- **Read cache:** 1-second monotonic cache for the raw `brightness` read, mirroring `playos_power_get_info()`.
- **Write authority:** preferred path is the trusted shell writing the sysfs node directly through the platform-api helper (the shell already opens `/dev/input/event*` directly and registers as a trusted shell). If the device-side check shows the shell's uid cannot write `brightness`, fall back to a `SetBrightness` IPC message owned by `playos-init` (root), mirroring `SetPerfProfile`.
- **UI step size:** 5% per d-pad press on the brightness row. No repeat-hold handling this sprint (shell edge detection already gives one press per edge).
- **Persistence:** out of scope. A brightness change is a sysfs write (survives until reboot but not across reboot). Boot-time restore of a saved level is a follow-up.

---

## Scope

### In Scope

- `playos_display.h` — add the two brightness functions.
- `playos_display.c` — sysfs backlight enumeration, read, clamp, and write.
- `playos-shell` — cache brightness in `shell.h`; add an interactive Brightness gauge to the Settings → Display tab; wire d-pad up/down to adjust and write; bump the Display tab's content height.
- `playos-spec` — update the brightness stub wording in `playos-shell-spec.md`; add this sprint doc.
- Verification: native compile check of platform-api + shell, QEMU boot where possible, and an Ally on-device brightness test.

### Explicitly Out of Scope

- Overlay volume/brightness HUD (`playos-spec/src/architecture.md:177`) — separate follow-up.
- Persisting brightness across reboot / boot-time restore of a saved level.
- Brightness hotkeys (volume/quick-menu button combos).
- HDR/adaptive-brightness/ambient-light sensor logic.
- Auto-dimming or thermal-driven backlight reduction.
- Intel/ultrabook-specific tuning beyond the fallback node preference.

---

## Required Repository Changes

| Repo | Required work |
|---|---|
| `playos-platform-api` | Add `playos_display_get_brightness()` / `playos_display_set_brightness()` to `playos_display.h`; implement sysfs backlight access in `playos_display.c` |
| `playos-shell` | Add brightness state to `include/shell.h`; add interactive gauge row to `src/screen_settings.c`; wire d-pad up/down adjustment + write; bump `TAB_DISPLAY` content height |
| `playos-spec` | Add `Sprint-9.5.md`; update the "display brightness (stub)" line in `src/playos-shell-spec.md` |

*(Conditional — only if the device-side check blocks direct writes)*

| Repo | Required work |
|---|---|
| `playos-refdistro` (`src/playos-init`) | Add a `SetBrightness` IPC handler that writes `/sys/class/backlight/<node>/brightness` as root |
| `playos-runtime` | Add the `SetBrightness` message type to the runtime IPC protocol |

---

## Expected Files and Directories

### `playos-platform-api`

```text
include/playos/playos_display.h   # add get/set brightness prototypes
src/playos_display.c              # replace stub with sysfs backlight implementation
```

### `playos-shell`

```text
include/shell.h                   # add display brightness cache + optional display cursor state
src/screen_settings.c             # add Brightness gauge row + d-pad adjustment on Display tab
```

### `playos-spec`

```text
src/sprints/Sprint-9.5.md         # this document
src/playos-shell-spec.md          # update brightness stub wording
```

---

## Agent Task Breakdown

### Task Status Grid

| Task ID | Task | Primary repo | Status | Notes / evidence |
|---|---|---|---|---|
| S9.5-T1 | Verify the Ally backlight node and write permission | `playos-refdistro` (on-device) | deferred | direct-write path chosen; node name + writability still unconfirmed on hardware |
| S9.5-T2 | Implement brightness get/set in `playos_display.c` | `playos-platform-api` | done | sysfs enumeration + 1 s cache + clamp; native build clean |
| S9.5-T3 | Add interactive Brightness gauge to Settings → Display | `playos-shell` | done | gauge row + d-pad up/down write + height bump; native build clean |
| S9.5-T4 | (Conditional) `SetBrightness` IPC fallback | `playos-refdistro`, `playos-runtime` | not needed | direct sysfs write chosen; T1 hardware check deferred |
| S9.5-T5 | Spec/docs reconciliation | `playos-spec` | done | shell-spec stub wording updated |
| S9.5-T6 | Build + validation | `playos-platform-api`, `playos-shell`, `playos-refdistro` | in progress | native compile clean; QEMU + Ally test pending |

---

### S9.5-T1 — Verify the Ally backlight node and write permission

**Finding:** `CONFIG_DRM_AMDGPU=y` + `CONFIG_DRM_AMD_DC=y` (`playos-refdistro/br2-external/board/ally/linux.config:96-100`) imply the amdgpu DC backlight node should exist at `/sys/class/backlight/amdgpu_bl0/`. This is not yet confirmed from the build tree or any captured USB log (the last mounted log dirs were empty).

**Steps:**

1. On the Ally (or from a shell on the mounted USB rootfs):
   - `ls -1 /sys/class/backlight/` → confirm `amdgpu_bl0` (or note the actual node name).
   - `cat /sys/class/backlight/<node>/max_brightness` and `cat .../brightness`.
   - `ls -l /sys/class/backlight/<node>/brightness` → note owner/group/mode.
   - `id` of the running `playos-shell` process.
2. Record the exact node name, max value, and whether the shell's uid can write the node.

**Done when:** the report names the concrete backlight node and states whether the trusted shell can write `brightness` directly. This gates T4.

---

### S9.5-T2 — Implement brightness get/set in `playos_display.c`

**Finding:** `playos-platform-api/src/playos_display.c` is a stub returning -1 for everything. `playos_display.h` currently has `PlayOSDisplayInfo`, `playos_display_get_info()`, and `playos_display_set_vsync()`.

**Steps:**

1. In `playos_display.h`, add:
   - `int playos_display_get_brightness(int *percent);`
   - `int playos_display_set_brightness(int percent);`
   with doc comments matching the existing style.
2. In `playos_display.c`, add a `read_int_file()` helper (copy the pattern from `playos_power.c:50-61`) and a backlight enumeration helper:
   - `opendir("/sys/class/backlight")`
   - for each entry, read `max_brightness`; prefer `amdgpu_bl0`, then `acpi_video0`, then `intel_backlight`, then the first non-`acpi_` entry with `max_brightness > 0`.
   - cache the chosen node path + `max_brightness` on first success.
3. Implement `playos_display_get_brightness()`:
   - read `brightness` with the 1-second monotonic cache (mirror `playos_power_get_info()`'s `g_cached_valid`/`g_cached_ms` style);
   - scale to 0..100 and return 0; return -1 if no node exists.
4. Implement `playos_display_set_brightness()`:
   - clamp `percent` to 0..100;
   - scale to raw using the cached `max_brightness`;
   - `fopen` the `brightness` node in write mode, write the integer + `"\n"`, `fflush`, `fclose`;
   - return 0 on success, -1 on failure (log once via `PLAYOS_LOG_W` on the failure, not every attempt).
5. Keep `playos_display_get_info()` and `playos_display_set_vsync()` stubs untouched unless a real implementation is trivially available — this sprint only adds brightness.

**Done when:** the platform-api library compiles natively and the functions read/write the expected sysfs node in a unit-style check (or report clean -1 when the node is absent).

---

### S9.5-T3 — Add interactive Brightness gauge to Settings → Display

**Finding:** the Display tab draws only Resolution / DPI Scale / GPU (`playos-shell/src/screen_settings.c:616-632`). Its update path is the generic read-only scroll branch (`screen_settings.c:218-226`). `settings_content_height()` returns `3 * info_h` for Display (`screen_settings.c:110-113`). `draw_trigger_gauge()` already renders a horizontal value bar (`screen_settings.c:375`).

**Steps:**

1. In `include/shell.h`, add cached brightness fields next to the power-info block (`shell.h:114-115`), e.g.:
   - `int display_brightness;` (0..100, -1 when unknown)
   - `int display_brightness_valid;`
   Optionally add `int settings_display_cursor;` if the row needs focus; otherwise d-pad up/down on the Display tab directly adjusts brightness.
2. Refresh brightness once per frame or on a short interval in the Settings update path (call `playos_display_get_brightness()`).
3. In `screen_settings.c` Display tab update (before the generic scroll branch), when `s->settings_tab == TAB_DISPLAY`:
   - d-pad up → `playos_display_set_brightness(current + 5)`, update cache;
   - d-pad down → `playos_display_set_brightness(current - 5)`, update cache.
   Keep tab switching on d-pad left/right unchanged.
4. In the Display tab draw block, add a "Brightness" row after GPU:
   - render the label via `draw_info_line()`-style layout;
   - render the gauge with `draw_trigger_gauge()` (or an equivalent horizontal bar) at `value = percent / 100.0f`;
   - show `NN%` as the value label.
5. Bump `settings_content_height()` for `TAB_DISPLAY` from `3.0f * info_h` to `4.0f * info_h` (`screen_settings.c:110-113`).
6. Ensure the write is idempotent and safe: if `playos_display_get_brightness()` returns -1 (no node), draw "Brightness — Unavailable" and no-op on up/down.

**Done when:** the Display tab shows a Brightness gauge whose value tracks the real backlight, and d-pad up/down changes both the gauge and the panel brightness.

---

### S9.5-T4 — (Conditional) `SetBrightness` IPC fallback

**Finding:** the shell already writes nothing to sysfs today; its privilege level for `/sys/class/backlight/<node>/brightness` is unverified. `playos-init` is root and already owns sysfs writes for performance profiles.

**Steps (only if T1 shows the shell cannot write directly):**

1. Add a `SetBrightness` message type to `playos-init/ipc/ipc.h` and the runtime IPC protocol.
2. In `playos-init`, add a handler that validates `0..100`, scales to `max_brightness`, and writes `/sys/class/backlight/<node>/brightness`.
3. In `playos_display_set_brightness()`, route through the IPC control socket (`/run/playos/control.sock`), mirroring `request_profile_over_ipc()` in `playos_power.c:234-314`.
4. Emit nothing back to games this sprint; the shell reads back via `playos_display_get_brightness()`.

**Done when:** `SetBrightness` changes the backlight and the shell gauge reflects it, with init as the sole sysfs writer.

---

### S9.5-T5 — Spec/docs reconciliation

**Steps:**

1. Update `playos-spec/src/playos-shell-spec.md:47` so brightness is no longer described as a stub — state that Settings → Display exposes a live brightness control backed by the platform-api.
2. Add `Sprint-9.5.md` to the sprint directory (this document).
3. Update cross-links if needed: the Sprint 9 and Sprint 10 footers can be adjusted to reference Sprint 9.5 where it sits chronologically (optional, low priority).

**Done when:** no spec text describes brightness as an unimplemented stub.

---

### S9.5-T6 — Build + validation

**Steps:**

1. Native compile-check `playos-platform-api` (the new `playos_display.c`) and `playos-shell` against the platform-api headers.
2. Build a QEMU image (or at least the shell + platform-api) to confirm no link regressions; brightness will report "Unavailable" in QEMU since there is no backlight node — that is the expected path.
3. On the Ally: open Settings → Display, confirm the gauge shows the current level, press up/down and verify the panel brightness changes and the percentage label tracks.
4. Capture evidence: `/data/log/shell.log` (or the platform-api log channel) showing the chosen backlight node and the successful get/set path.

**Done when:** native + QEMU builds are clean, and the Ally Settings → Display brightness control changes the panel backlight.

---

## Implementation Guidance

### Order of execution

1. **T1 first** (device verification) — decides whether T4 is needed.
2. **T2 second** (platform-api) — the primitive everything else depends on.
3. **T3 third** (shell UI) — surfaces the primitive.
4. **T4 fourth** (IPC fallback) — only if T1 blocks direct writes.
5. **T5 fifth** (docs) — document reality after the code lands.
6. **T6 last** (validation) — build + on-device.

### Atomic commits

```
S9.5-T2: add display brightness get/set to platform-api
S9.5-T3: add brightness gauge to settings display tab
S9.5-T5: update spec for display brightness control
```

### Keep it additive

- No public header is removed or reordered; `playos_display.h` only gains two functions.
- `PLAYOS_API_VERSION` is **not** bumped (additive functions only, no struct/ABI change).
- The shell's read-only tab behaviour is preserved for Audio / Power / Network / Input; only Display gains the new interactive row.

---

## Verification and Evidence

| Evidence | How it is produced |
|---|---|
| Backlight node confirmed | T1 output: `ls /sys/class/backlight/`, `max_brightness`, writability, shell uid |
| API works | native test / on-device `playos_display_get_brightness()` matches `cat .../brightness` scaled to % |
| Write works | on-device `playos_display_set_brightness(50)` changes `cat .../brightness` |
| UI works | Settings → Display gauge tracks real value; up/down changes the panel |
| No-op path safe | QEMU (no backlight) shows "Unavailable" and does not crash |
| Docs updated | `playos-shell-spec.md` no longer calls brightness a stub |

---

## Acceptance Criteria

- [ ] `playos_display_get_brightness()` returns the real 0..100 level on the Ally
- [ ] `playos_display_set_brightness()` writes the scaled value to the sysfs backlight node
- [ ] Settings → Display shows a Brightness gauge with a percentage label
- [ ] D-pad up/down on the Display tab adjusts brightness and writes it immediately
- [ ] Display tab content height fits the new row without clipping
- [ ] The no-backlight path (QEMU / no node) reports unavailable and does not crash
- [x] `PLAYOS_API_VERSION` is unchanged; the change is additive
- [x] `playos-shell-spec.md` no longer describes brightness as a stub
- [ ] Native and QEMU builds are clean

---

## Handoff to Sprint 10

Sprint 10 may assume:

- `playos_display.h` exposes a working `playos_display_get_brightness()` / `playos_display_set_brightness()` pair.
- The shell Settings → Display tab can read and adjust panel brightness.
- Brightness writes go through the platform-api helper (direct sysfs, or init-owned IPC if T4 was needed).
- Persistence across reboot, brightness hotkeys, and the overlay HUD remain unimplemented and are natural follow-ups.

---

## Exit Gate

The ROG Ally's panel backlight can be read and adjusted from Settings → Display, and the change is reflected live through the platform-api brightness primitive.

*Previous: [Sprint 9](Sprint-9.md) | Next: [Sprint 10](Sprint-10.md)*
