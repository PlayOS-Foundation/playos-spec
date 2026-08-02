# Sprint 8 — ALSA Audio

**Goal:** Integrate reliable ALSA audio into the Raylib PlayOS backend. Games and the shell can play audio through the ROG Ally's built-in speakers and headphones. Audio behaves correctly across lifecycle transitions.

**Primary Outcome:** `com.playos.sample-audio` plays a looping sine tone from the ROG Ally speakers. Audio pauses when the game is backgrounded, resumes when foregrounded, and stops cleanly when the game exits.

**Prerequisites:** Sprint 7 complete — full console lifecycle working (background/foreground events delivered).

---

## Why This Sprint Exists

Sprint 7 delivers a working console lifecycle but with no audio. Games on a gaming device must have audio. Sprint 8 wires ALSA directly into the Raylib backend already established in `rcore_playos.c`, making audio a first-class feature that works correctly across all lifecycle transitions.

---

## Start Condition Checklist

- Sprint 7 complete: full lifecycle (launch/background/resume/quit/crash) working on the Ally.
- `rcore_playos.c` exists in `playos-shell` from Sprint 5.
- `playos-overlay` exists with volume control placeholder from Sprint 7.
- ROG Ally audio hardware confirmed working in Linux (ALSA recognizes the device).

---

## Decisions Locked for This Sprint

- **ALSA only:** no PulseAudio, no PipeWire, no SDL audio; direct ALSA PCM in `rcore_playos.c`
- **Sample rate:** 44100 Hz (document this; do not change without an ADR)
- **Format:** signed 16-bit little-endian stereo (`SND_PCM_FORMAT_S16_LE`, 2 channels)
- **Volume ownership:** system-wide; only the shell/overlay may call `playos_audio_set_master_volume()`; games cannot change master volume
- **Device priority:** `PLAYOS_AUDIO_DEVICE` env var → headphone jack → built-in speakers
- **Audio thread policy:** `SCHED_FIFO` or `SCHED_RR` with a safe priority; document the chosen value

---

## Scope

### In Scope

- ALSA PCM backend in `rcore_playos.c`
- `playos_audio.h` public API
- Audio lifecycle behavior (pause on BACKGROUND, resume on FOREGROUND, stop on TERMINATE)
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
| `playos-platform-api` | `rcore_playos.c` ALSA backend, `playos_audio.h` |
| `playos-shell` | Shell audio (startup/UI sounds) |
| `playos-overlay` | Volume display + D-pad control |
| `playos-refdistro` | ALSA libs in Buildroot config, `com.playos.sample-audio` package |
| `playos-spec` | Audio policy doc (lifecycle behavior, volume model) |

---

## Expected Files and Directories

### `playos-platform-api`

```text
include/playos/playos_audio.h
src/rcore_playos.c           # ALSA backend (already exists from Sprint 5; extend here)
src/playos_audio.c           # playos_audio_get_info, set_master_volume, set_muted
```

### `playos-refdistro`

```text
br2-external/board/common/rootfs-overlay/data/games/com.playos.sample-audio/
    manifest.json
    bin/audio-sample
```

---

## Agent Task Breakdown

### Task Status Grid

| Task ID | Task | Primary repo | Status | Notes / evidence |
|---|---|---|---|---|
| S8-T1 | Implement ALSA PCM backend in `rcore_playos.c` | `playos-platform-api` | not started | |
| S8-T2 | Define and implement `playos_audio.h` API | `playos-platform-api` | not started | |
| S8-T3 | Implement audio lifecycle behavior (background/foreground/terminate) | `playos-platform-api` | not started | |
| S8-T4 | Implement headphone jack detection and routing | `playos-platform-api` | not started | |
| S8-T5 | Add volume control to `playos-overlay` | `playos-overlay` | not started | |
| S8-T6 | Add shell UI sounds | `playos-shell` | not started | |
| S8-T7 | Build `com.playos.sample-audio` sample game | `playos-refdistro` | not started | |
| S8-T8 | Audio validation on Ally | `playos-refdistro` | not started | |

### S8-T1 — Implement ALSA PCM backend in `rcore_playos.c`

**Initialization:**
1. `snd_pcm_open()` on the resolved device name (see device priority)
2. If default fails: enumerate with `snd_device_name_hint()`, select first stereo hardware output
3. Hardware params: 2 ch, `SND_PCM_FORMAT_S16_LE`, 44100 Hz, buffer/period sized for stable playback
4. Software params: set start threshold, avail-min
5. Prepare PCM handle; start the audio thread

**Audio thread:**
- `SCHED_FIFO` or `SCHED_RR` at a documented priority
- Call Raylib audio mixing callback
- Write to ALSA via `snd_pcm_writei()`
- On `EPIPE` (underrun): `snd_pcm_prepare()` + continue
- On `ESTRPIPE` (suspend): wait and retry

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

Volume and mute are system-wide; only the shell/overlay may call the setters. Games may call `playos_audio_get_info()` to read current state.

**Done when:** API compiles, `playos_audio_get_info()` returns a populated struct, and `set_master_volume(0.5f)` produces an audible volume change.

### S8-T3 — Implement audio lifecycle behavior

| Lifecycle event | Audio action |
|---|---|
| `PLAYOS_LIFECYCLE_FOREGROUND` | Resume ALSA playback at previous volume |
| `PLAYOS_LIFECYCLE_BACKGROUND` | Drain buffer; block writes (thread pauses) |
| `PLAYOS_LIFECYCLE_TERMINATE` | Stop thread; `snd_pcm_close()` |

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

### S8-T7 — Build `com.playos.sample-audio` sample game

- Generates a 440 Hz sine wave via Raylib audio API
- Plays as a looping 2-second buffer
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

Tune buffer size and period size empirically on the Ally. Start with `buffer_size = 4096 frames`, `period_size = 1024 frames`. Adjust if underruns occur. Document the final values in a comment in `rcore_playos.c`.

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

Replace any SDL or non-ALSA audio path with a direct ALSA PCM implementation.

**Initialization:**
1. Call `snd_pcm_open()` — open the default playback device (`default` or a PlayOS-configured device name)
2. If the default fails, enumerate with `snd_device_name_hint()` and select the first hardware stereo output
3. Set hardware parameters: stereo (2 ch), signed 16-bit little-endian, 44100 Hz (or 48000 Hz — pick one and document)
4. Set software parameters: buffer size, period size (tune for stable low-latency playback)
5. Prepare the PCM handle; start the audio thread

**Audio thread:**
- Runs at real-time or high priority (use `sched_setscheduler(SCHED_FIFO)` or `SCHED_RR` with a safe priority)
- Calls the Raylib audio mixing callback
- Writes mixed PCM to ALSA via `snd_pcm_writei()`
- Handles `EPIPE` (underrun): call `snd_pcm_prepare()` and continue
- Handles `ESTRPIPE` (suspend): wait and retry

**Device selection priority:**
1. `PLAYOS_AUDIO_DEVICE` environment variable (for testing)
2. Headphone jack if plugged in (detect via ALSA mixer or `/proc/asound/`)
3. Built-in speakers

**Device change handling:**
- Monitor ALSA for device add/remove (use inotify on `/dev/snd/` or ALSA's async notification)
- On headphone plug/unplug: close old PCM, open new device, resume audio stream seamlessly (or with brief gap)

### `playos-platform-api` — Audio API

Define `include/playos/playos_audio.h`:

```c
typedef struct {
    int     sample_rate;     /* e.g. 44100 or 48000 */
    int     channels;        /* 1 or 2 */
    int     bits_per_sample; /* 16 */
    float   master_volume;   /* 0.0 – 1.0 */
    int     muted;
} PlayOSAudioInfo;

/* Get current audio device info. Returns 0 on success. */
int playos_audio_get_info(PlayOSAudioInfo *info);

/* Set master volume [0.0, 1.0]. */
int playos_audio_set_master_volume(float volume);

/* Mute/unmute. */
int playos_audio_set_muted(int muted);
```

Volume and mute are system-wide and controlled by PlayOS (shell and overlay), not by individual games.

### Audio Lifecycle Behavior

Implement the following policy in the Raylib PlayOS backend:

| Lifecycle Event | Audio action |
|---|---|
| `PLAYOS_LIFECYCLE_FOREGROUND` | Resume ALSA playback at previous volume |
| `PLAYOS_LIFECYCLE_BACKGROUND` | Pause ALSA thread (drain buffer, then block writes) |
| `PLAYOS_LIFECYCLE_TERMINATE` | Stop ALSA thread, close PCM handle |

**Shell audio behavior:**
- Shell plays UI sounds (button press feedback) while it is the foreground client
- When game becomes foreground, shell stops its audio stream
- When game backgrounds or exits, shell resumes audio

**No PulseAudio or PipeWire required.** If future multi-application mixing is needed, a `playos-audio` service is introduced (post-MVP).

### Volume Control in Overlay

Add volume control to `playos-overlay`:
- Show current volume percentage
- D-pad Up/Down adjusts volume (calls `playos_audio_set_master_volume()`)
- Mute/unmute toggle (L1/R1 or a dedicated button)
- Volume change is reflected immediately (no lag)

### `com.playos.sample-audio` — Audio Sample Game

A minimal sample game that:
- Generates a 440 Hz sine wave using Raylib's audio API
- Plays it as a looping 2-second buffer
- Shows frequency and volume on screen
- Responds to lifecycle events (stops when backgrounded)
- Uses `playos_audio_get_info()` to display device info

Add this sample to the test image in `playos-refdistro`.

---

## Acceptance Criteria

- [ ] Shell plays audio (a short startup chime or UI click sounds) on the Ally speakers
- [ ] `com.playos.sample-audio` plays a continuous sine tone through built-in speakers
- [ ] Plugging in headphones routes audio to headphones
- [ ] Unplugging headphones routes audio back to speakers (may have brief gap)
- [ ] System button → overlay: game audio stops (or mutes) within 200ms
- [ ] Overlay "Resume": game audio resumes within 200ms
- [ ] Quit game: audio stops; shell audio resumes if applicable
- [ ] No audio underruns during normal gameplay (< 1 per 60 seconds is acceptable)
- [ ] `playos_audio_set_master_volume(0.5f)` sets volume to 50%; audible change confirmed
- [ ] `playos_audio_set_muted(1)` silences all audio
- [ ] Volume overlay shows current level; D-pad adjusts it
- [ ] CI passes (QEMU: audio backend compiles; no runtime audio test in CI)

---

## Repositories Primarily Involved

| Repo | Work |
|---|---|
| `playos-platform-api` | ALSA backend in `rcore_playos.c`, `playos_audio.h` API |
| `playos-shell` | Shell audio integration (startup sound, UI clicks) |
| `playos-overlay` | Volume/mute controls |
| `playos-refdistro` | ALSA libs in Buildroot, `com.playos.sample-audio` package |
| `playos-spec` | Audio policy document (one owner, lifecycle behavior, volume model) |

---

## Testing Approach

- Physical ROG Ally required for all audio tests
- Automated: boot, play sine sample, measure for underruns over 2 minutes
- Headphone detection: plug/unplug USB-C audio adapter or 3.5mm; verify routing
- Lifecycle: launch sample-audio, press System button, verify silence; resume, verify audio
- Volume: test min/max/mid settings; verify no clipping at max
- QEMU CI: audio library compiles, ALSA backend initializes without crashing (device open will fail gracefully)

---

## Exit Gate

Games and the shell play audio on the ROG Ally. Audio transitions correctly across all lifecycle events. Volume is controlled through the overlay.

*Previous: [Sprint 7](Sprint-7.md) | Next: [Sprint 9](Sprint-9.md)*
