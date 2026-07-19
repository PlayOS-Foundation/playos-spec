# Use wlroots as the PlayOS Compositor Base

- **ADR:** 0002
- **Title:** Use wlroots as the PlayOS Compositor Base
- **Status:** Accepted
- **Date:** 2026-01-01
- **Last reviewed:** 2026-07-19
- **Deciders:** PlayOS Foundation

## Context

PlayOS Runtime Devices need a dedicated presentation subsystem that behaves like a console rather than a desktop. It must establish the display early during boot, keep the PlayOS Shell available, present one active game fullscreen, place trusted system UI above that game, enforce focus and input policy, and restore the shell immediately when a game exits or crashes.

The compositor must provide or integrate:

- direct ownership of DRM/KMS outputs,
- display mode, transform, scaling, refresh-rate, and hotplug policy,
- Wayland buffer presentation,
- keyboard, pointer, touch, and controller-derived navigation routing,
- shell, game, overlay, and trusted system UI layers,
- Wayland-native clients and, where justified, XWayland compatibility,
- presentation timing and a path to direct scanout.

The PlayOS Shell is not the compositor. The shell, games, overlays, and trusted system UI are replaceable Wayland clients.

The compositor is also not the application supervisor. Launching processes, managing cgroups and permissions, detecting exits and crashes, and coordinating suspend and resume belong to PlayOS runtime services such as `playos-sessiond`. The compositor owns presentation, surface classification, layer ordering, output policy, and input focus.

The guiding principle is:

> **Borrow infrastructure. Own the policy.**

Three base approaches were evaluated:

1. **Gamescope.** Use Gamescope as the display engine and run the PlayOS Shell as a client. Gamescope provides mature gaming features such as scaling, frame limiting, HDR, VRR, overlays, and compatibility behaviour. However, adopting it as the compositor would make PlayOS presentation policy depend on an external project and its release cadence.

2. **Bare-metal Raylib DRM.** Let Raylib using `PLATFORM_DRM` own DRM/KMS directly. This is small and gives direct control, but it would require PlayOS to rebuild multi-client presentation, overlay layering, input focus, output hotplug, compatibility support, and switching infrastructure.

3. **A dedicated PlayOS compositor on wlroots.** wlroots supplies backend and session management, DRM/KMS integration, buffer allocation, rendering integration, libinput, output management, Wayland protocol infrastructure, and optional XWayland support. PlayOS implements the console-specific policy above it.

TinyWL is a useful reference for initial wlroots bring-up, but it is an example compositor rather than a suitable long-term PlayOS architecture.

## Decision

PlayOS will implement a **dedicated, console-focused Wayland compositor in C++ on wlroots**.

TinyWL will be used only as **initial scaffolding and a learning reference**. PlayOS will progressively replace TinyWL example policy with explicit PlayOS modules, protocols, tests, and lifecycle integration. The lasting architectural dependency is wlroots, not TinyWL.

The compositor will:

- own DRM/KMS and the presentation path,
- keep the shell alive as the background client,
- present at most one active game in the game layer,
- maintain trusted overlay and system UI layers above the game,
- classify surfaces and enforce PlayOS focus and input policy,
- restore the existing shell when a game exits, crashes, or is terminated,
- report output and presentation state to runtime services,
- support direct scanout when safe and beneficial,
- support XWayland only where compatibility requirements justify it.

Initial implementations may use `xdg-shell` and `wlr-layer-shell`, with runtime IPC associating clients with PlayOS roles. The long-term design may introduce private, versioned protocols for shell, game, overlay, trusted system UI, and compositor control roles.

PlayOS may borrow implementation ideas selectively:

- **Cage** for a small fullscreen-first policy,
- **Gamescope** for gaming presentation behaviour,
- **Sway** for mature wlroots integration patterns,
- **Weston** for testing discipline.

These projects are references, not the owners of PlayOS policy.

The compositor remains within the `playos-runtime` repository under `compositor/` until independent release cadence, stable private protocols, dedicated CI, a hardware test matrix, and a versioned IPC contract justify a separate repository.

## Responsibility Boundary

The compositor is responsible for:

- output acquisition and configuration,
- client buffer presentation,
- surface roles and classification,
- layer ordering,
- visual activation of the shell or game,
- trusted overlay placement,
- input focus,
- presentation state and metrics.

Runtime services are responsible for:

- launching and supervising the shell and games,
- process groups and cgroups,
- permissions and isolation,
- exit and crash detection,
- package execution,
- suspend and resume coordination,
- requesting presentation transitions.

The compositor is not a package manager, game launcher, account service, entitlement service, or general-purpose desktop window manager.

## Staged Evolution

### Stage 1 — Vertical slice

- DRM/KMS bring-up,
- fullscreen PlayOS Shell,
- keyboard and controller routing,
- one launched game,
- immediate return to the shell on exit.

### Stage 2 — Runtime integration

- stable compositor/runtime IPC,
- explicit surface roles,
- Home-button handling independent of game responsiveness,
- pointer and touch support,
- trusted overlays,
- crash recovery,
- XWayland if required.

### Stage 3 — Console services

- quick menu and virtual keyboard,
- performance HUD,
- brightness and volume overlays,
- suspend and resume integration,
- docked and external-display handling.

### Stage 4 — Measured optimisations

- direct scanout,
- presentation statistics,
- frame limiting and scaling,
- refresh-rate policy,
- VRR, HDR, and colour-management evaluation.

## Consequences

### Positive

- PlayOS owns console presentation and focus policy.
- wlroots avoids reimplementing DRM/KMS, session, input, output, and Wayland infrastructure.
- The shell stays replaceable and independent from the compositor.
- Clear compositor/runtime boundaries keep process supervision out of the display subsystem.
- The architecture supports trusted overlays, crash recovery, multiple input classes, and future private protocols.
- TinyWL provides a small bootstrap path without constraining the final design.
- Gaming-specific optimisations can be added when measurements justify them.

### Negative

- PlayOS must implement and maintain game switching, trusted layering, focus policy, private protocols, and runtime IPC.
- wlroots APIs evolve and require deliberate version tracking and upgrades.
- Direct scanout, frame pacing, VRR, HDR, colour management, and suspend/resume need substantial hardware testing.
- A custom compositor requires more engineering and validation than adopting Gamescope unchanged.
- XWayland, if enabled, expands compatibility and security scope.

### Risks and Mitigations

- **TinyWL architecture leaks into production:** treat TinyWL as disposable scaffolding and replace example policy with explicit modules and tests.
- **Runtime/compositor responsibility drift:** keep application lifecycle in runtime services and presentation/focus in the compositor.
- **wlroots churn:** pin supported versions, isolate wlroots integration, and test upgrades deliberately.
- **Hardware-specific regressions:** maintain automated headless/nested tests plus a handheld and dock hardware matrix.
- **Feature creep into a desktop compositor:** preserve the fullscreen console model and explicit non-goals.

## Alternatives Considered

- **Gamescope as the PlayOS compositor:** rejected as the default architecture because PlayOS would not fully own its presentation policy or release timeline. Gamescope remains an important source of gaming compositor concepts and may still be useful in experiments or compatibility paths.
- **Bare-metal Raylib DRM:** rejected for general Runtime Devices because it would require rebuilding multi-client presentation, input focus, overlays, output management, and switching. It may remain suitable for constrained SDK targets or embedded devices that do not require the full PlayOS Runtime compositor model.
- **Writing the entire compositor stack without wlroots:** rejected because it duplicates mature low-level infrastructure without a demonstrated benefit.
- **Using TinyWL as the long-term compositor architecture:** rejected. TinyWL is a bring-up reference, not the PlayOS policy or modular design.
- **Using Cage unchanged:** rejected because its fullscreen policy is instructive but does not provide PlayOS-specific roles, trusted layers, lifecycle integration, or console presentation policy.

## Non-goals

This decision does not authorize the compositor to:

- become a general Linux desktop,
- support arbitrary workspaces or unrestricted overlapping windows,
- own application lifecycle or package management,
- expose wlroots as part of the public PlayOS API,
- accept unrestricted client overlays,
- remove wlroots without measured technical justification.

## References

- [Compositor Model](../book/src/08-runtime-architecture/05-compositor-model.md)
- [Boot Model](../book/src/08-runtime-architecture/03-boot-model.md)
- [Application Lifecycle](../book/src/08-runtime-architecture/06-application-lifecycle.md)
- [Runtime IPC](../book/src/08-runtime-architecture/09-runtime-ipc.md)
- [Input Routing](../book/src/08-runtime-architecture/10-input-routing.md)
- [Game Switching](../book/src/08-runtime-architecture/11-game-switching.md)
- [Overlay Integration](../book/src/08-runtime-architecture/12-overlay-integration.md)
