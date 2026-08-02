# ADR-0004 — wlroots as Compositor Foundation

**Date:** Sprint 2  
**Status:** Accepted  
**Deciders:** PlayOS core team

---

## Context

PlayOS needs a Wayland compositor. Options: implement from scratch using raw libwayland, use wlroots, use a full compositor (Sway, Wayfire) as a base.

## Decision

Use wlroots as the foundation for `playos-compositor`. wlroots provides mechanisms; PlayOS supplies console policy.

## Rationale

- **DRM/KMS abstraction:** wlroots handles the complex DRM backend initialization, output management, and KMS atomicity
- **Proven:** wlroots is used in production compositors (Sway, Hyprland, Cage) — it is well-tested on real hardware including AMD GPUs
- **Not a full compositor:** wlroots is a library, not a complete compositor — PlayOS defines all policy (focus, z-order, trusted roles, state machine) on top of it
- **Active maintenance:** Regular upstream development; AMD and Intel DRM paths are kept up to date
- **ROG Ally compatibility:** Sway and Cage are known to work on the ROG Ally's AMDGPU — wlroots underlying both is validated hardware

## Alternatives Considered

| Option | Rejected because |
|---|---|
| Raw libwayland | Would require reimplementing all DRM/KMS, buffer management, and Wayland protocol infrastructure that wlroots provides — massive scope increase |
| Sway as a base | Sway is a tiling window manager; its policy (floating/tiled windows, workspaces) would have to be removed entirely — more work than starting from wlroots |
| Wayfire | Plugin architecture is more complex than needed; wlroots is a cleaner starting point |
| Mutter / KWin | Heavy GNOME/KDE dependencies; incompatible with musl and minimal initramfs |

## Guiding Rule

> wlroots implements mechanisms; `playos-compositor` implements console policy; Raylib renders what the player sees.

## Consequences

- wlroots version must be pinned in `versions.lock`
- wlroots API is not stable across versions — compositor code may need updates when wlroots is bumped
- Private PlayOS Wayland protocol is layered on top of wlroots, not replacing its core protocols
