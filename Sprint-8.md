# Sprint 8 — ALSA Audio

**Goal:** Integrate reliable ALSA audio into the Raylib PlayOS backend. Games and the shell can play audio through the ROG Ally's built-in speakers and headphones. Audio behaves correctly across lifecycle transitions.

**Primary Outcome:** `com.playos.sample-audio` plays a looping sine tone from the ROG Ally speakers. Audio pauses when the game is backgrounded, resumes when foregrounded, and stops cleanly when the game exits.

**Prerequisites:** Sprint 7 complete — full console lifecycle working (background/foreground events delivered).

---

## Key Deliverables

### ALSA PCM Backend in Raylib (`rcore_playos.c`)

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
