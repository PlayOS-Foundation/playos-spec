# Sprint 8 — ALSA Audio

**Goal:** Integrate reliable ALSA audio into the Raylib PlayOS backend. Games and the shell can play audio through the ROG Ally's built-in speakers and headphones. Audio behaves correctly across lifecycle transitions.

**Primary Outcome:** `com.playos.sample-audio` plays a looping sine tone from the ROG Ally speakers. Audio pauses when the game is backgrounded, resumes when foregrounded, and stops cleanly when the game exits.

**Prerequisites:** Sprint 7 complete — full console lifecycle working (background/foreground events delivered).

---

## Why This Sprint Exists

Sprint 7 delivers a working console lifecycle but with no audio. Games on a gaming device must have audio. Sprint 8 enables ALSA audio through Raylib's miniaudio backend and implements the system audio controls, making audio a first-class feature that works correctly across all lifecycle transitions.

---

## Start Condition Checklist

- Sprint 7 complete: full lifecycle (launch/background/resume/quit/crash) working on the Ally.
- Raylib 6.0 vendored in `playos-shell`; miniaudio audio module present but disabled (`SUPPORT_MODULE_RAUDIO=0`).
- `playos_audio.h`/`playos_audio.c` stub exists in `playos-platform-api`.
- `com.playos.sample-audio` placeholder exists in `playos-samples/audio-sine`.
- `playos-overlay` source lives in `playos-refdistro/src/playos-overlay/` (Sprint 7).
- ROG Ally audio hardware confirmed working in Linux (ALSA recognizes the device).

---

## Decisions Locked for This Sprint

- **ALSA only:** no PulseAudio, no PipeWire, no SDL audio; enable Raylib's miniaudio ALSA backend (`raudio.c` → `miniaudio.h`, vendored in `playos-shell`) and link `alsa-lib`. Raylib/miniaudio has no PipeWire backend (it supports only ALSA, PulseAudio, JACK), so this is the default — but compile out PulseAudio with `MA_NO_PULSEAUDIO` (alongside the existing `MA_NO_JACK`) to guarantee only ALSA is built in.
- **Sample rate:** 44100 Hz (document this; do not change without an ADR)
- **Format:** signed 16-bit little-endian stereo (`SND_PCM_FORMAT_S16_LE`, 2 channels)
- **Volume ownership:** system-wide. The shell/overlay owns the volume UI and is always honored; a game's `playos_audio_set_master_volume()`/`set_muted()` request is honored only while that game is foreground. Games may always read state via `playos_audio_get_info()`.
- **Device priority:** `PLAYOS_AUDIO_DEVICE` env var → headphone jack → built-in speakers
- **Audio thread policy:** miniaudio owns the mixing thread; if a real-time class is needed use `SCHED_FIFO`/`SCHED_RR` at a safe priority and document the value

---

## Scope

### In Scope

- ALSA PCM backend via Raylib's miniaudio module (`raudio.c`), enabled in `playos-shell`
- `playos_audio.h` public API — implement the existing `playos-platform-api` stub
- Audio lifecycle behavior (pause on BACKGROUND/SUSPEND, resume on FOREGROUND/RESUME, stop on TERMINATE)
- Headphone jack detection and routing switch
- Volume control in overlay
- `com.playos.sample-audio` sample game
- Shell UI sounds (startup chime or click feedback)

### Explicitly Out of Scope

- PulseAudio / PipeWire integration (post-MVP)
- HDMI audio (post-MVP)
- Bluetooth audio (post-MVP)
- Per-game volume settings (post-MVP)
- Multi-application audio mixing service (post-MVP)

---

## Required Repository Changes

| Repo | Required work |
|---|---|
| `playos-shell` | Enable Raylib miniaudio ALSA backend (`SUPPORT_MODULE_RAUDIO=1`); shell UI sounds |
| `playos-platform-api` | Implement `playos_audio.c` (system state, master volume/mute) — replace stub |
| `playos-refdistro` | `alsa-lib` in Buildroot config; overlay volume control in `src/playos-overlay/` |
| `playos-samples` | Finish `com.playos.sample-audio` (actual sine playback) |
| `playos-spec` | Audio policy doc (lifecycle behavior, volume model) |

---

## Expected Files and Directories

### `playos-platform-api`

```text
include/playos/playos_audio.h   # exists (stub declarations) — finalize contract
src/playos_audio.c              # implement playos_audio_get_info, set_master_volume, set_muted
```

### `playos-shell`

```text
external/raylib/src/config.h    # SUPPORT_MODULE_RAUDIO 0 → 1
external/raylib/src/raudio.c    # miniaudio ALSA backend (enable + link alsa-lib)
src/*.c                         # shell UI sounds (startup chime, navigation clicks)
```

### `playos-refdistro`

```text
br2-external/package/playos-raylib/        # link alsa-lib
src/playos-overlay/                        # volume display + D-pad control
```

### `playos-samples`

```text
audio-sine/src/main.c                      # actual 440 Hz sine playback (placeholder today)
audio-sine/manifest.json                   # already installed as com.playos.sample-audio
```

---

## Agent Task Breakdown

### Task Status Grid

| Task ID | Task | Primary repo | Status | Notes / evidence |
|---|---|---|---|---|
| S8-T1 | Enable ALSA audio via Raylib miniaudio backend | `playos-shell` | not started | `SUPPORT_MODULE_RAUDIO` currently 0 |
| S8-T2 | Define and implement `playos_audio.h` API | `playos-platform-api` | in progress | Stub header + `playos_audio.c` exist; finalize contract + implement |
| S8-T3 | Implement audio lifecycle behavior (background/foreground/suspend/resume/terminate) | `playos-platform-api` | not started | |
| S8-T4 | Implement headphone jack detection and routing | `playos-platform-api` | not started | |
| S8-T5 | Add volume control to `playos-overlay` | `playos-refdistro` | not started | Lives in `src/playos-overlay/` |
| S8-T6 | Add shell UI sounds | `playos-shell` | not started | |
| S8-T7 | Build `com.playos.sample-audio` sample game | `playos-samples` | in progress | Placeholder exists (`audio-sine`); actual sine playback pending |
| S8-T8 | Audio validation on Ally | `playos-refdistro` | not started | |

### S8-T1 — Enable the ALSA audio backend in Raylib's miniaudio module

**Enablement:**
1. Set `SUPPORT_MODULE_RAUDIO=1` in `playos-shell`'s vendored Raylib build; add `alsa-lib` to the Buildroot target. Also `#define MA_NO_PULSEAUDIO` next to the existing `MA_NO_JACK` in `raudio.c` so miniaudio compiles only the ALSA backend.
2. Confirm miniaudio's ALSA backend opens the default playback device (`default`, or `PLAYOS_AUDIO_DEVICE` if set).
3. If the default device fails: enumerate with `snd_device_name_hint()` and select the first stereo hardware output.
4. Params: 44100 Hz, stereo `SND_PCM_FORMAT_S16_LE` (miniaudio resamples to the device native rate/format as needed).
5. Prepare and start playback; miniaudio owns the mixing thread.

**Audio thread / underrun handling:**
- Set a documented `SCHED_FIFO`/`SCHED_RR` priority only if a real-time class is needed.
- Handle underrun (`EPIPE`) and suspend (`ESTRPIPE`) recovery; log device switches.

**Done when:** `com.playos.sample-audio` produces audible output from the built-in speakers.

### S8-T2 — Define and implement `playos_audio.h` API

```c
typedef struct {
    int   sample_rate;
    int   channels;
    int   bits_per_sample;
    float master_volume;   /* 0.0 – 1.0 */
    int   muted;
} PlayOSAudioInfo;

int playos_audio_get_info(PlayOSAudioInfo *info);
int playos_audio_set_master_volume(float volume);
int playos_audio_set_muted(int muted);
```

Volume and mute are system-wide. The shell/overlay owns the volume UI and its calls are always honored. A game's setter calls are requests, honored only while the game is foreground; games may always call `playos_audio_get_info()` to read state.

**Done when:** API compiles, `playos_audio_get_info()` returns a populated struct, and `set_master_volume(0.5f)` produces an audible volume change.

### S8-T3 — Implement audio lifecycle behavior

| Lifecycle event | Audio action |
|---|---|
| `PLAYOS_LIFECYCLE_FOREGROUND` | Resume ALSA playback at previous volume |
| `PLAYOS_LIFECYCLE_BACKGROUND` | Drain buffer; block writes (thread pauses) |
| `PLAYOS_LIFECYCLE_SUSPEND` | Pause playback (same as background; flush before returning) |
| `PLAYOS_LIFECYCLE_RESUME` | Resume playback |
| `PLAYOS_LIFECYCLE_TERMINATE` | Stop thread; close PCM handle |

Shell behavior: while shell is foreground, its audio is active; when game starts, shell audio pauses; when game exits, shell audio resumes.

**Done when:** lifecycle test — launch audio sample, press system button (audio stops within 200ms), resume (audio restarts within 200ms), quit (audio stops cleanly).

### S8-T4 — Implement headphone jack detection and routing

- Monitor ALSA mixer or inotify on `/dev/snd/` for device add/remove
- On headphone plug: close current PCM; reopen on headphone device; resume stream
- On headphone unplug: close headphone PCM; reopen on speakers; resume stream (brief gap acceptable)
- Log all device switches

**Done when:** plugging a USB-C audio adapter (or 3.5mm) routes audio to it; unplugging returns audio to speakers.

### S8-T5 — Add volume control to `playos-overlay`

- Display current volume as a percentage bar
- D-pad Up/Down adjusts in 5% steps via `playos_audio_set_master_volume()`
- L1 or dedicated button toggles mute via `playos_audio_set_muted()`
- Volume changes take effect immediately (no lag)

**Done when:** D-pad adjusts volume in the overlay and the change is audible in real-time.

### S8-T6 — Add shell UI sounds

- Short startup chime on shell launch
- Click/confirm sound on menu navigation and selection
- Error sound on rejected actions
- All sounds implemented via Raylib audio API using the ALSA backend

**Done when:** navigating the shell produces audible feedback sounds.

### S8-T7 — Finish `com.playos.sample-audio` sample game

- Finish the existing `playos-samples/audio-sine` placeholder (already installed as `com.playos.sample-audio`)
- Generates a 440 Hz sine wave via Raylib audio API, played as a looping 2-second buffer
- Shows on-screen: frequency, volume, device name, current lifecycle state
- Responds to lifecycle events: pauses sine tone when backgrounded
- Uses `playos_audio_get_info()` to display device info

**Done when:** game appears in shell library, plays the sine tone, and the on-screen info is accurate.

### S8-T8 — Audio validation on Ally

- Confirm built-in speakers produce output
- Measure underrun count over 2 minutes of continuous playback (target: 0; ≤ 1/min acceptable)
- Headphone plug/unplug routing test (×3 cycles)
- Lifecycle transition test: background → silence; resume → audio; quit → stop
- Volume: min/max/mid; verify no clipping at max
- QEMU CI: ALSA backend compiles; PCM open fails gracefully with log; no crash

**Done when:** all validation cases pass and evidence is logged.

---

## Implementation Guidance

### ALSA period sizing

Tune miniaudio's buffer and period size empirically on the Ally (Raylib exposes the device buffer/period size in `config.h`). Start with a small buffer and increase if underruns occur. Document the final values.

### SIGSTOP and audio

If the compositor sends `SIGSTOP` to a non-cooperative game, the audio thread is also stopped. This is acceptable behavior — the OS enforces silence automatically.

### CI audio

The ALSA backend must compile in CI. `snd_pcm_open()` will fail in a headless VM — this is expected. Log the failure with device name and continue; do not assert.

---

## Verification and Evidence

| Evidence | How it is produced |
|---|---|
| Speaker output | Confirmed by ear on the Ally |
| Underrun count | ALSA `snd_pcm_status` after 2-minute run |
| Headphone routing | Plug/unplug ×3, confirm routing in log |
| Lifecycle timing | Timestamp of BACKGROUND event vs. first silent frame (≤200ms) |
| Volume API | `playos_audio_get_info()` output before and after `set_master_volume()` |
| CI build | CI log showing successful compile |

---

## Acceptance Criteria

- [ ] Shell plays audio (UI sounds) on the Ally speakers
- [ ] `com.playos.sample-audio` plays a continuous sine tone through built-in speakers
- [ ] Plugging in headphones routes audio to headphones
- [ ] Unplugging headphones routes audio back to speakers (brief gap acceptable)
- [ ] System button → overlay: game audio stops within 200ms
- [ ] Overlay "Resume": game audio restarts within 200ms
- [ ] Quit game: audio stops; shell audio resumes
- [ ] No audio underruns during normal playback (< 1/min acceptable)
- [ ] `playos_audio_set_master_volume(0.5f)` produces an audible change
- [ ] `playos_audio_set_muted(1)` silences all audio
- [ ] Volume overlay shows current level; D-pad adjusts it
- [ ] CI passes (ALSA backend compiles; PCM open failure is handled gracefully)

---

## Handoff to Sprint 9

Sprint 9 may assume:

- Audio is fully functional and lifecycle-aware
- `playos-overlay` can be extended with new status displays and controls
- `playos-init` thermal monitoring loop can be added without conflicting with audio
- The full lifecycle including `SIGSTOP`/`SIGCONT` is stable

---

## Exit Gate

Games and the shell play audio on the ROG Ally. Audio transitions correctly across all lifecycle events. Volume is controlled through the overlay.

*Previous: [Sprint 7](Sprint-7.md) | Next: [Sprint 9](Sprint-9.md)*
