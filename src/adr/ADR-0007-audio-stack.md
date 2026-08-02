# ADR-0007 — Direct ALSA for MVP Audio

**Date:** Sprint 8  
**Status:** Accepted  
**Deciders:** PlayOS core team

---

## Context

PlayOS needs audio output. Options: PipeWire, PulseAudio, ALSA directly, a custom audio mixer.

## Decision

Use ALSA PCM directly for the MVP. No PipeWire or PulseAudio in v0.1.0. A dedicated audio service is introduced post-MVP only when simultaneous mixing becomes necessary.

## Rationale

- **MVP scope:** The MVP requires one foreground audio owner at a time (game while foreground, shell otherwise). This maps perfectly to a single exclusive ALSA PCM device — no mixer needed.
- **Minimal dependencies:** ALSA is built into every Linux system; no extra userspace daemon required in the initramfs
- **Simpler lifecycle:** Audio ownership follows the compositor lifecycle directly — game gets ALSA handle on foreground, releases it on background
- **PipeWire complexity:** PipeWire adds ~5MB to the image, requires a session manager, and introduces a latency and complexity budget that is not justified for one audio owner
- **ROG Ally ALSA:** The ROG Ally's AMD ACP audio works with ALSA; `snd_soc_acp*` drivers are mature

## Limitations Accepted

- Only one process can play audio at a time
- No system notification sounds over game audio
- No Bluetooth audio (post-MVP, requires audio service)
- No per-application volume (system volume only)

## Migration Path

When post-MVP features require simultaneous audio (notifications over games, Bluetooth audio), a `playos-audio` service is introduced. The service:
- Owns the ALSA device exclusively
- Exposes a simple IPC for shell, overlay, and game audio streams
- Handles mixing in userspace

This migration does not break the `playos_audio.h` public API — the backend changes, the API stays the same.

## Consequences

- The Raylib PlayOS backend must implement a direct ALSA PCM path
- Device selection, underrun handling, and headphone detection are implemented in `libplayos`
- Shell audio and game audio cannot overlap in the MVP
