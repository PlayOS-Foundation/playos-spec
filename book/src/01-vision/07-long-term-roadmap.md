# Long-Term Roadmap

PlayOS development follows a phased approach from proof-of-concept to a
stable 1.0 release. Each phase adds capability while preserving the
compatibility guarantees established in earlier phases.

## Phase 0 — Prototype (current)

**Goal:** Prove the concept with a working vertical slice.

- Platform API with Lifecycle, Input, Capabilities, Storage modules.
- Linux Runtime with wlroots compositor.
- Reference Shell (Raylib-based) with Home screen, library, and game
  launching.
- Reference device support: ROG Ally (primary), Generic Linux PC.
- Sample application (`hello-playos`).

**Status:** In progress. Core modules implemented, shell functional,
ROG Ally input mapping working.

## Phase 1 — Platform Foundation

**Goal:** Complete the core Platform API and stabilize the Runtime.

- Complete remaining Platform API modules (Audio, Display, Network,
  Battery, Power, Bluetooth).
- Stabilize the Runtime's package execution, game switching, and
  suspend/resume.
- Device profiles for all reference devices.
- Certification test suite (API conformance).
- Developer CLI (`playos-tools`).

## Phase 2 — Cloud and Marketplace

**Goal:** Ship the cloud services and marketplace.

- Self-hostable cloud stack (accounts, package storage, catalog).
- Marketplace with store browsing, purchase, and download.
- Cloud saves, leaderboards, and achievements.
- Developer portal for publishing and managing applications.

## Phase 3 — SDK Targets

**Goal:** Expand the developer surface.

- Windows SDK Target (complete).
- macOS SDK Target.
- Android SDK Target.
- PS Vita SDK Target.
- Engine integration kits (SDL, Godot).

## Phase 4 — 1.0

**Goal:** Stable release.

- All Platform API modules stable at 1.0.
- All reference devices certified.
- Official marketplace live.
- Documentation complete.
- Governance model active.

## Beyond 1.0

- Additional certified devices from OEM partners.
- Additional SDK Targets (WebAssembly, iOS).
- Additional engine integrations.
- Federation between stores.
- Streaming and remote play.

## Related

- [Specification Status](../00-front-matter/02-status.md)
- [Release Process](../16-governance-and-process/09-release-process.md)
