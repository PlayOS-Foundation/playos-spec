# Use wlroots/TinyWL as the PlayOS Compositor Base

- **ADR:** 0002
- **Title:** Use wlroots/TinyWL as the PlayOS Compositor Base
- **Status:** Accepted
- **Date:** 2026-01-01
- **Deciders:** PlayOS Foundation

## Context

PlayOS needs a display compositor for its Runtime Devices. The compositor must:

- own DRM/KMS for direct GPU rendering,
- manage fullscreen application surfaces,
- route input (controllers, touch, keyboard, mouse),
- support game switching (shell ↔ game ↔ shell),
- support future overlay UI,
- integrate with the Raylib-based reference shell.

Three options were evaluated:

1. **Gamescope (Valve).** Treat Gamescope as a "display engine /
   firmware" and build the PlayOS shell as a Wayland client on top.
   Gamescope already solves game switching, resolution changes, FPS
   limiting, HDR, VRR, screenshots, overlay, and suspend/resume.
   Downside: PlayOS does not own the compositor; Valve does. This
   limits customisation and ties the platform to Valve's release
   cadence.

2. **Bare-metal Raylib DRM.** Raylib in `PLATFORM_DRM` owns KMS/DRM
   directly. The PlayOS shell renders directly to the display with no
   compositor. Launching games requires process management (fork/exec,
   systemd scopes, SIGSTOP/SIGCONT, cgroups). This gives maximum
   control but requires reimplementing game switching, overlay,
   input routing, output hotplug, resolution changes, and
   suspend/resume from scratch.

3. **Custom wlroots compositor, starting from TinyWL.** wlroots
   provides DRM/KMS backend, libinput integration, Wayland protocol
   handling, output management, and XWayland support. TinyWL is the
   canonical minimal wlroots compositor (~1000 lines of C). PlayOS
   would own and evolve its own compositor, written in C++ with RAII
   wrappers around wlroots objects.

## Decision

PlayOS will build its **own compositor based on wlroots**, starting
from the TinyWL reference and evolving it through staged milestones.

The compositor will be written in **C++** (wlroots is C but works
correctly from C++). The Raylib shell will run as a **Wayland client**
on this compositor. The compositor owns KMS/DRM.

**Staged evolution:**

- **Stage 1:** TinyWL + fullscreen-only + controller input support.
- **Stage 2:** PlayOS compositor with launcher IPC, Home button
  handling, game switching, overlay surface, XWayland support.
- **Stage 3:** Full console OS integration — Raylib XMB shell,
  Steam/RetroArch/Heroic launching, performance HUD, suspend/resume,
  brightness/volume management.

## Consequences

**Positive:**
- PlayOS owns its entire display stack — no dependency on Valve's
  Gamescope release cycle.
- wlroots provides mature, battle-tested implementations of DRM/KMS,
  libinput, Wayland protocol handling, and output management.
- TinyWL is small enough to understand completely and evolve
  incrementally.
- C++ RAII wrappers align with the existing PlayOS architectural
  patterns.
- Multiple fullscreen surfaces, overlay, and XWayland are supported
  by the wlroots model.

**Negative:**
- More initial development effort than choosing Gamescope.
- PlayOS is responsible for game switching, overlay, and
  suspend/resume — these are not free.
- wlroots API may change across versions; the compositor must track
  upstream.

## Alternatives Considered

- **Gamescope:** Rejected because PlayOS would not control its own
  display stack or release timeline.
- **Bare-metal Raylib DRM:** Rejected because it requires
  reimplementing too much infrastructure (input routing, output
  hotplug, overlay, game switching) that wlroots already provides.
  However, this remains a possible fallback for minimal/embedded
  devices that do not need multi-surface compositing.
- **Writing a compositor from scratch (without wlroots):** Rejected as
  significantly more effort with no clear benefit over wlroots.
