# Audio API

The `PlayOS::Audio` module provides master volume and mute control. It is
part of the core (portable) subset per RFC-0004: audio output control that
every conformant target can support.

## API

```cpp
namespace PlayOS {
namespace Audio {

float GetMasterVolume();           // range [0.0, 1.0]
void  SetMasterVolume(float vol);  // clamped to [0.0, 1.0]
bool  IsMuted();                   // true if audio output is muted
void  SetMuted(bool muted);        // mute or unmute

} // namespace Audio
} // namespace PlayOS
```

## Usage

```cpp
// Read current volume.
float vol = PlayOS::Audio::GetMasterVolume();

// Set volume to 50%.
PlayOS::Audio::SetMasterVolume(0.5f);

// Toggle mute.
if (PlayOS::Audio::IsMuted()) {
    PlayOS::Audio::SetMuted(false);
}
```

## Scope

This module controls **master** volume — the system-wide audio output level
on the device. It does not provide:

- Per-channel or per-stream volume (that is the engine's responsibility).
- Audio playback or streaming (that is the engine's responsibility).
- Spatial audio or surround configuration (these are optional capabilities
  tracked under `audio.spatial` — see [Capability Groups](../05-capability-model/07-capability-groups.md)).

## Capability

The master volume API is part of the core subset and does not require a
separate capability query. Optional audio features (spatial audio,
microphone, advanced mixing) are gated by their own capabilities.

## Requirements

- `SetMasterVolume` MUST clamp input to `[0.0, 1.0]`.
- `GetMasterVolume` MUST return the last value set by `SetMasterVolume`,
  or the system default at startup.
- Mute state MUST be independent of volume level: un-muting MUST restore
  the volume set before mute.
- On devices without a hardware mixer, the implementation SHOULD be a
  no-op that reports success.

## Implementation note

The reference implementation (`playos/audio.h`) delegates to a
platform-specific `IAudioBackend`. The Raylib backend uses
`SetMasterVolume`/`IsAudioDeviceReady`. The null backend returns a fixed
volume of `1.0` (no hardware mixer present).

## Related

- [API Design Rules](02-api-design-rules.md)
- [Audio Porting](../12-device-model-and-porting/06-audio-porting.md)
