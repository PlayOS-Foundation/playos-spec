# Game Audio

PlayOS owns the audio device through ALSA. Games mix their own streams with
their audio framework (e.g., Raylib audio). `libplayos` exposes system-wide
audio state and volume control.

## API

```c
PlayOSAudioInfo info;
if (playos_audio_get_info(&info) == 0) {
    // sample_rate, channels, bits_per_sample, master_volume, muted
}

playos_audio_set_master_volume(0.8f);  // only when foreground
playos_audio_set_muted(1);             // only when foreground
```

## Rules

- Volume/mute requests are honored only while the game is foreground.
- Pause/mute your own audio on `PLAYOS_LIFECYCLE_BACKGROUND`.
- Use the system master volume sparingly — respect the player's setting.
