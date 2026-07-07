# Founding Conversation Summary

**Source:** ChatGPT conversation, exported as `notes/chat-raw.txt`.
**Date:** Mid 2026.
**Status:** Historical record — the PlayOS Book and RFCs are the
authoritative source of truth.

## Key decisions recorded here

### Branding
- Internal codename "GameOS" → public brand **PlayOS**.
- Domain: **playos.dev**.
- GitHub org: **PlayOS-Foundation**.
- Motto: **"Write Once. Play Anywhere. Own Everything."**

### Platform identity
- PlayOS is an **open console platform specification**, not only a
  Linux distribution or a launcher.
- **Specification-first:** the PlayOS Book is the source of truth;
  implementations follow the spec.

### Display / compositor
- **Decision:** Build a custom compositor on **wlroots**, starting
  from **TinyWL**, written in **C++** with RAII wrappers.
- Raylib shell runs as a **Wayland client** on this compositor.
- **Rejected:** Gamescope (too dependent on Valve) and bare-metal
  Raylib DRM (too much infrastructure to reinvent).
- See [ADR-0002](../adr/0002-use-wlroots-tinywl-compositor.md).

### Reference runtime
- **Decision:** **Arch Linux** base, with `archiso`/`mkarchiso` for
  reproducible ISO images.
- Boot path: `UEFI → kernel → systemd → seatd → PlayOS compositor → shell`.
- Critical boot: GPU → input → shell. Audio, Wi-Fi, Bluetooth start
  async.
- See [ADR-0003](../adr/0003-arch-linux-reference-runtime-base.md).

### Device model
- **Runtime Devices** (full stack): ROG Ally, x86_64 PC, Orange Pi 5,
  future handhelds.
- **SDK Targets** (API subset): Windows, macOS, Linux desktop,
  Android, PS Vita homebrew via VitaSDK.

### Engine strategy
- **Engine-agnostic.** Raylib is the reference engine and shell
  technology, not a hard dependency.
- Platform API must work with SDL, Godot, custom engines.
- The layer stack: `Game → Engine (Raylib/SDL/Godot/Custom) → PlayOS Platform API → Runtime → OS → Hardware`.

### Capability model
- `PlayOS::Capabilities::Has(Capability::Touch)` instead of
  `#ifdef ANDROID`.
- Backend architecture: `IInputBackend`, `IPowerBackend`,
  `IDisplayBackend` interfaces with per-platform implementations.
- Device profiles (TOML) map hardware-specific controls.

### Package format
- `.gpk` packages: `manifest.json + executable + assets/ + icon.png + signature`.
- Self-contained, signed, with permissions.

### Marketplace & cloud
- **Self-hostable by design.** Multi-tenant: official, community,
  OEM, private, LAN stores.
- SDK-first marketplace: developers can publish from Windows without
  owning a PlayOS device.
- Backend: API + PostgreSQL + MinIO/S3 + Redis + Keycloak/Authentik +
  Prometheus + Grafana.

### Business model
- Don't sell the OS — build the ecosystem.
- Revenue: hosted marketplace commission (5-10%), developer
  subscriptions (analytics, cloud saves, crash reporting, CDN), OEM
  licensing and white-label builds.

### Governance
- **Stage 1 (company-led):** rapid iteration.
- **Stage 2 (PlayOS Foundation):** owns SDK spec, API, certification,
  documentation.
- **Stage 3 (multi-vendor):** hardware vendors participate.
- Open Platform Promise: games written against a stable API continue
  to run on future compatible implementations.

### Repository structure
`playos-spec`, `playos-platform-api`, `playos-runtime`,
`playos-shell`, `playos-cloud`, `playos-marketplace`,
`playos-tools`, `playos-samples`, `playos-reference-devices`,
`playos-foundation`.

### Agent workflow
- RFCs as Markdown files; GitHub Issues as the work queue.
- `AGENTS.md` + `.github/copilot-instructions.md` per repo.
- Labels: `rfc`, `spec`, `docs`, `agent-ready`, `needs-review`,
  `accepted`, `implementation-ready`.
