# Sprint 14.5 — Shell-Owned Install Progress

**Goal:** make the install experience stay inside PlayOS from confirm to completion. Today the shell owns the front-end (disk list, hold-A confirm), then hands the screen to a separate installer app for the destructive phase. This sprint keeps the *same* engine and the *same* handoff guarantees, but the shell draws the progress, the completion and any error — no second fullscreen app, no visible seam.

**Primary Outcome:** pressing confirm on the installer screen leaves the shell's own UI (same fonts, colours, layout) on screen for the whole install; progress advances step by step with a percentage; completion offers **Reboot now**; a failed install shows an error card in the shell and the session stays usable. The standalone installer still works unchanged for `playos.mode=install`, `playos.install.auto=1` and recovery.

**Status:** **4 of 5 tasks done** (T1–T4 complete; T5's happy path verified on hardware). Two verification checks are deliberately parked — see "Parked" below. Recorded 2026-09-13 from the S14-T10 close-out; verified on device 2026-09-22.

**Prerequisites:** S14-T10 complete and verified on hardware (install 8/8 steps on the ROG Ally, seamless handoff, failure returns to the shell); S13.7 live-USB/installer images in place.

---

## Why This Sprint Exists

S14-T10 removed the *behavioural* seam (the session no longer tears down, the installer is styled like the shell, the console is suppressed) but not the *visual* one: the shell's screen is replaced by the installer process for the destructive phase, and the shell is stopped while it runs. That is a workable fallback — it is also why the installer exists as a standalone program — but it means the most consequential thing a user can do on the device happens in a different program than the one they were using.

Keeping the presentation in the shell is small because everything else already exists: the step engine, the IPC, the supervision model, and the styling.

## Start Condition Checklist

- [x] Install engine exists in `playos-refdistro/src/playos-installer/` (`main.c`, `format.c`, `efi.c`) and is verified on hardware.
- [x] `StartInstaller` IPC with `target_disk` and `PLAYOS_INSTALL_TARGET` forwarding works (S14-T10).
- [x] init supervises children with a restart policy and can start/stop them without touching the compositor (S14-T10 handoff).
- [x] The shell already renders progress-shaped UI for the *update* path (`update_in_progress` → `draw_update_progress_row`), so the widgets and the state handling are proven.
- [x] The trusted IPC has an event stream the shell already consumes (`UpdateProgress`), so a second progress channel follows an existing pattern.
- [x] Confirmed 2026-09-20 (re-alignment pass): **no `format.c`/`efi.c` entry point needs a caller-supplied
  callback.** They already take `(device, partno, label, err, errlen)` and return an int, so they are
  library-ready as they stand. The step *dispatch* is `main.c:run_install_step()`, and that is what T1 has
  to lift: it currently takes the whole UI `struct installer`, so it needs a slim context (target device,
  step index, step name, error buffer) plus one **step callback** (begin / end / error) that the worker
  turns into `InstallProgress`. The only place that would want finer granularity is the payload write
  (`playos_format_write_image`, a ~55 MB copy) - a byte-level callback there is optional, since a per-step
  percentage (i/8) is enough for the first version.

## Decisions Locked for This Sprint

- **One engine, two front-ends.** The install logic is extracted into `libplayos-install` (static, built in `src/playos-installer/`) and used by *both* the standalone installer and the new worker. No forked logic, no second implementation.
- **No new socket.** Progress travels over the existing trusted control socket: a `PrepareInstall` request and `InstallProgress` events, mirroring `ApplyUpdate`/`UpdateProgress`.
- **init owns lifecycle, the shell owns presentation.** init spawns the worker exactly as it spawns the shell/overlay, and is the only component that talks to it; the shell never sees the worker.
- **The worker must be screen-less.** It never creates a Wayland surface. If the shell is not running, `StartInstaller` falls back to the standalone installer (existing behaviour) rather than starting a worker nobody can see.
- **Step names and log lines stay identical** to today's installer so existing logs, docs and any test comparing them keep working.
- **No partition-layout or A/B changes.** This sprint is presentation + plumbing only.

## Scope

**In scope:** the engine extraction; `PrepareInstall` / `InstallProgress` IPC (init + runtime wrapper + spec); the `playos-install-worker` package; the shell's progress, completion and error screens; keeping the standalone installer path intact; on-device verification of all three paths (shell-driven install, forced failure, standalone installer image).

**Out of scope:** A/B layout or `boot.json` semantics; dm-verity; the `.playosb` update path (already shell-owned); the installer's disk picker and hold-to-confirm (S14-T10, unchanged); the 11.5 `wipefs` follow-up (tracked separately — see `09-next-steps.md`).

## Required Repository Changes

- `playos-refdistro` — `src/playos-installer/`: split the engine into a static `libplayos-install`, add `worker.c` (the new `playos-install-worker` binary), keep `main.c` as the standalone front-end; extend the package `.mk`; update `scripts/gen-*-usb-image.sh` if the worker must ship in the live image.
- `playos-init` — `PLAYOS_IPC_TYPE_PREPARE_INSTALL` handling, worker spawn/stop with the existing supervision patterns, and `InstallProgress` emission (or the worker posts directly to the shell listener — decide in T2).
- `playos-runtime` — `playos_trusted_prepare_install()` wrapper and the progress event type, matching the existing trusted API style.
- `playos-shell` — replace the "handing off" transition with a progress screen, add the completion and error cards, and keep `SCREEN_INSTALLER` as the single owner of the flow.
- `playos-spec` — document the new IPC types in `runtime-ipc.md`, the worker in `playos-installer-spec.md`, and the shell screens in `playos-shell-spec.md`.
- `playos-memorymap` — status/evidence; close the follow-up entry that this sprint exists to address.

## Expected Files and Directories

```
playos-refdistro/src/playos-installer/
├── install_lib.c / install_lib.h   ← engine extracted from format.c + efi.c
├── worker.c                        ← playos-install-worker (no UI)
├── main.c                          ← standalone installer (unchanged behaviour)
└── CMakeLists.txt                  ← libplayos-install + two binaries
playos-shell/src/screen_installer.c ← progress / complete / error stages
```

## Agent Task Breakdown

## Parked (revisit later)

Decided 2026-09-22: the shell-driven install works end to end on hardware, so these
two checks are parked rather than blocking Sprint 15. Neither is expected to fail —
both exercise paths already proven in other forms — but neither has been run on the
device, and this sprint's acceptance line names the first one explicitly.

**P1 — forced failure (killed worker → error card).** Needs the USB stick booted
live and an install started (a couple of button presses). Then, over SSH:

```
kill $(pgrep -f playos-install-worker)
```

Expect the shell to show the **error card** with init's reason
(`install worker exited (code=… signal=…)`), holding on screen for at least
`INSTALL_CARD_HOLD_OFF`, instead of a progress bar that never advances or a silent
return to the shell. Evidence to collect: `/data/log/shell-stderr.log`
(`install error: …`), `/data/log/init.log` (`install worker exited`), and a
screenshot of the card (tap COMMAND).

**P2 — standalone installer rerun.** Needs the stick live (its `playos-a` carries
the payload). Then:

```
PLAYOS_INSTALL_TARGET=/dev/nvme0n1 /usr/bin/playos-installer
```

The picker is skipped when the target is set. Expect the same eight steps and the
same log lines as the worker produces (`installer step N/8 …`), proving the engine
extraction did not change that front-end's behaviour, and its own UI on screen. It
reboots on success, so the SSH session ends — that is expected.

### Task Status Grid

| Task ID | Task | Primary repo | Status | Notes / evidence |
|---|---|---|---|---|
| S14.5-T1 | Extract `libplayos-install` from the installer's step engine | `playos-refdistro` | done (device check with T5) | Behaviour-preserving: the standalone installer must still pass its on-device run  `libplayos-install` extracted (`install_lib.[ch]`, static lib, installer links it) and building; main.c still uses its own copy of the step switch - rewiring that is the rest of T1.  `libplayos-install` extracted and building; `main.c` drives it through a thin wrapper with log + seed-key callbacks; the engine owns every format/EFI call and the step table has one definition. On-device "behaves identically" confirmation rides with T5. |
| S14.5-T2 | `PrepareInstall` + `InstallProgress` IPC types, trusted wrapper, spec | `playos-init`, `playos-runtime`, `playos-spec` | done (plumbing) | Mirrors `ApplyUpdate`/`UpdateProgress`  6 IPC types; `playos_trusted_prepare_install()`; init validates the target (absent / holds `/` or `/data` -> error) and releases its `/EFI`; progress events relayed to the shell listener like `UpdateProgress`; documented in `runtime-ipc.md`. The sprint's "shell drives a full install" done-when is exercised by T3/T4. |
| S14.5-T3 | `playos-install-worker` package: runs the engine, reports progress, no surface | `playos-refdistro`, `playos-init` | done | Supervised like shell/overlay; no restart mid-write  `worker.c` + `playos-install-worker` (NEEDED = libfdisk + libc only); runtime progress/complete/error reporters; init mounts `/mnt/payload`, spawns the worker keeping shell/overlay/compositor/SSH up, falls back to the standalone handoff with no shell listener, and reports a dead worker as `InstallError`. init builds clean, 5/5 tests. The sprint named a `playos-installer-spec.md` that did not exist - it does now. |
| S14.5-T4 | Shell draws progress, completion ("Reboot now") and errors | `playos-shell` | done (device check with T5) | Reuses the update-progress widgets and `SCREEN_INSTALLER`  confirm -> `PrepareInstall` (reason-bearing errors) -> `StartInstaller` -> progress stage; four stages in `SCREEN_INSTALLER` (picker/confirm/progress/complete/error); completion offers "A: Reboot now"; `shell_poll_json` makes the event payload readable (the update gauge had been stuck at 0%% for the same reason). Builds clean. |
| S14.5-T5 | On-device verification: shell-driven install, forced failure, standalone path | `playos-refdistro` | in progress | Evidence in `playos-refdistro/docs/`  Happy path **verified on hardware** 2026-09-22: the shell stays on the installer screen, progress + percentage are visible, the success card appears and "A: Reboot now" boots the installed system (`/dev/nvme0n1p2`, `boot.json` `slot_a` good). Full evidence and the four hardware-only bugs it found: `playos-refdistro/docs/s14.5-install-verification-2026-09-22.md`. Still to verify: forced-failure error card, standalone-installer rerun. |

---

### S14.5-T1 — Extract the install engine

Turn the linear step sequence in `main.c` (`format` → `write payload` → `ESP`/`boot.json` → verify) into a callable engine with a small context struct and a progress callback. Keep every step name and log line. The standalone installer becomes a thin front-end over it.

**Done when:** `playos-installer` behaves identically on device, and the engine can be driven with a callback instead of `printf`.

### S14.5-T2 — IPC

Add `PrepareInstall` (target validation + mount release, so failures are detected *before* the shell commits to a progress screen) and `InstallProgress` events (step index, step count, percent, message, terminal ok/error). Decide and document whether the worker posts to the shell listener directly or via init; whichever is chosen, the shell must see a terminal event on every failure path, including a worker that dies.

**Done when:** the shell can drive a full install with no installer surface on screen, and a killed worker surfaces as an error rather than a stalled progress bar.

### S14.5-T3 — The worker

A screen-less binary that runs the engine and reports progress. Spawned by init with the same environment discipline as the shell (`XDG_RUNTIME_DIR`, log redirection to `/data/log/install-worker.log`). On failure it must not be restarted mid-write: exit, report, and leave recovery to the shell.

**Done when:** `playos-install-worker` completes an install on device with the shell visible throughout, and its log is readable after a failure.

### S14.5-T4 — Shell screens

Extend `SCREEN_INSTALLER` past the confirm stage: progress (step checklist + bar + percent, matching the installer's current look), completion (`Reboot now` → the existing `Reboot` IPC; also offer staying in the shell), and an error card naming the failing step and the log path. Keep the standalone-installer fallback when the shell is not running.

**Done when:** the flow from confirm to reboot never leaves the shell's UI, and the error card is reachable in a forced-failure test.

### S14.5-T5 — On-device verification

1. Full install from the dev USB image with the shell visible the whole way; capture screenshots of progress and completion.
2. Forced failure (e.g. keep a filesystem mounted on the target, or corrupt the payload) → error card, session usable afterwards.
3. Standalone path: boot with `playos.mode=install` (and `auto=1`) → the existing installer still runs.
4. Record evidence in `playos-refdistro/docs/` with the image hashes.

## Implementation Guidance

- **Do not re-implement the destructive steps.** If the engine needs a change, change it once and let both front-ends inherit it.
- **Progress must be honest.** Report the engine's real step boundaries; never animate a bar with a timer.
- **Every terminal path needs an event.** Worker exit without a terminal `InstallProgress` is a bug, not an edge case.
- **Keep the fallback alive.** The standalone installer is the recovery path when no shell is running; do not delete it to simplify this sprint.
- **Watch the write window.** The target's mounts must be released before the worker writes, and nothing else may touch that disk while it does — the S14-T10 handoff rules still apply, they are just now inside one session.

## Verification and Evidence

| Evidence | How it is produced | Current state |
|---|---|---|
| Shell stays on screen for the whole install | Screenshots at confirm, mid-progress and completion on the Ally | ⬜ not started |
| Progress advances per step | Worker log step names match the shell's displayed steps | ⬜ not started |
| Completion reboots into the new install | Post-reboot `boot.json` + slot contents | ⬜ not started |
| Failure is visible and non-fatal | Forced-failure test: error card + shell still usable | ⬜ not started |
| Standalone installer unaffected | `playos.mode=install` boot reaches the picker | ⬜ not started |
| No regression in the T10 guarantees | Compositor + `/data` stay up; only the target ESP released | ⬜ not started |

## Acceptance Criteria

- [ ] Confirming "Install PlayOS" never replaces the shell's UI with another program's surface.
- [ ] Progress shows the current step and a percentage, and advances at step boundaries.
- [ ] Completion offers "Reboot now" and the device boots the new install.
- [ ] A failed install shows an error card in the shell, names the failing step and the log path, and leaves the session usable.
- [ ] The standalone installer still performs an install for `playos.mode=install` / `auto=1`.
- [ ] Step names and log lines are unchanged from S14-T10.
