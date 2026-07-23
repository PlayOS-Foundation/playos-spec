# Audio Porting

The audio backend provides platform-wide volume and mute controls.

## Interface to implement

```cpp
// backends/audio_backend.h
class IAudioBackend {
public:
    virtual float GetMasterVolume() = 0;       // [0.0, 1.0]
    virtual void SetMasterVolume(float v) = 0; // Clamp to [0.0, 1.0]
    virtual bool IsMuted() = 0;
    virtual void SetMuted(bool muted) = 0;
};
```

This is system-level audio control — not per-app audio mixing. The
backend talks to the platform's audio infrastructure (PipeWire,
WASAPI, ALSA) to get and set the master volume.

## Linux implementation (PipeWire)

**Current gap: Linux audio backend does not exist.** The design is:

- **Primary path:** PipeWire pulse-compat API via `libpipewire` or
  the PulseAudio simple API. Most modern Linux desktops and the
  PlayOS compositor use PipeWire.
- **Fallback:** ALSA mixer (`snd_mixer_*` API) for minimal systems.
- **Device enumeration:** List audio sinks to detect headphones,
  speakers, and Bluetooth audio.

The backend should:
1. Connect to PipeWire/PulseAudio.
2. Watch for `sink-input` changes to track per-app volume (future).
3. Respond to hotplug events (headphones connected/disconnected).

## Windows implementation (WASAPI)

**Current gap: Windows audio backend does not exist.** The design is:

- **Primary path:** WASAPI `IAudioEndpointVolume` interface.
- **Mute:** `IAudioEndpointVolume::GetMute/SetMute`.
- **Device enumeration:** `IMMDeviceEnumerator`.

## Raylib implementation

The Raylib backend calls `GetMasterVolume()` and `SetMasterVolume()`
— Raylib's internal audio master that controls miniaudio output.
This works for SDK Targets on desktop OSes where Raylib manages all
audio.

## Null backend

Reports volume 1.0, not muted. All `Set*` calls are no-ops.

## Porting steps

1. Create `src/backends/<platform>/<platform>_audio_backend.cpp`.
2. Implement `IAudioBackend`.
3. Call `Capabilities::RegisterCapability(Capability::AudioOutput)` if the
   platform has audio hardware (most do).
4. Register in CMake.

## Implementation gaps

| Platform | Backend status | Notes |
|---|---|---|
| Linux | ❌ Missing | Need PipeWire/PulseAudio integration |
| Windows | ❌ Missing | Need WASAPI `IAudioEndpointVolume` |
| Raylib | ✅ Complete | `GetMasterVolume/SetMasterVolume` |
| Android | ❌ Missing | Need `AudioManager` integration |
| PS Vita | ❌ Missing | Need `sceAudio` integration |

## Related

- [Backend Porting Model](03-backend-porting-model.md)
- [Audio Module (Capability Model)](../05-capability-model/01-overview.md)
