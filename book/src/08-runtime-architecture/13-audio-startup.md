# Audio Startup

The Runtime enumerates audio devices at startup and makes them
available to applications through the Audio API.

## Startup sequence

1. Runtime queries the operating system for audio devices.
2. For each device found, creates an `AudioDeviceInfo` entry.
3. Registers capabilities based on what was found:
   - At least one output device → `audio.core` present.
   - Stereo output → `audio.stereo` present.
   - Surround output → `audio.surround` present.
4. Routes system audio (shell sounds, notifications) through the
   default device.
5. Makes device list available to applications.

## Device enumeration

```cpp
// Query available audio devices
int count = Audio::GetDeviceCount();
for (int i = 0; i < count; i++) {
    Audio::DeviceInfo info = Audio::GetDeviceInfo(i);
    // info.name, info.type (Output/Input), info.channels, info.sampleRate
}
```

## Default device selection

The Runtime selects the default audio device based on:

1. Device profile configuration.
2. Last-used device (persisted across boots).
3. System default (OS preference).

On handheld devices with built-in speakers, the built-in speakers are
the default. When headphones (wired or Bluetooth) are connected, the
Runtime switches audio to the headphones automatically.

## Device hot-plug

The Runtime detects audio device changes at runtime:

- **Headphones connected** → switch audio to headphones.
- **Headphones disconnected** → switch back to speakers.
- **Bluetooth audio connected** → switch to Bluetooth device.
- **New USB DAC** → make available, do not auto-switch.

Applications receive an `AudioDeviceChanged` event.

## Audio policy

- Only one application outputs audio at a time (the foreground app).
- Shell sounds (navigation, notifications) mix with application audio
  at reduced volume.
- When the overlay is open, application audio is ducked (reduced by
  50%).
- Suspended applications do not output audio.

## Related

- [Audio API](../06-platform-api/12-audio-api.md)
- [Runtime Services](08-runtime-services.md)
- [Overlay Integration](12-overlay-integration.md)
