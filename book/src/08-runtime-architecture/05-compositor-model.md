# Compositor Model

The PlayOS reference runtime uses a dedicated, console-focused Wayland compositor implemented in C++ on top of **wlroots**. The implementation began from the **TinyWL** reference compositor, but PlayOS treats TinyWL only as initial scaffolding rather than as the final architecture.

The compositor owns the display through DRM/KMS. The PlayOS Shell, games, and system overlays are Wayland clients. This decision is recorded in [ADR-0002](https://github.com/PlayOS-Foundation/playos-spec/blob/main/adr/0002-use-wlroots-tinywl-compositor.md).

> **Architecture principle:** Borrow infrastructure. Own the policy.

## 1. Purpose

The compositor is the presentation subsystem of the PlayOS Runtime. It is not a desktop environment and is not intended to become a general-purpose window manager.

Its purpose is to:

- establish the display as early as possible during boot,
- keep the PlayOS Shell continuously available,
- present one active game as the primary fullscreen surface,
- display trusted system UI above a running game,
- route input according to PlayOS focus and lifecycle policy,
- recover cleanly when a game exits or crashes,
- isolate hardware-specific display behaviour from applications.

## 2. Architectural position

```text
Linux kernel
    ├── DRM/KMS
    ├── evdev
    └── device drivers
          │
          ▼
wlroots
    ├── backend and session management
    ├── buffer allocation
    ├── rendering integration
    ├── libinput integration
    └── Wayland protocol infrastructure
          │
          ▼
PlayOS Compositor
    ├── output policy
    ├── surface classification
    ├── focus and input policy
    ├── layer ordering
    ├── shell/game switching
    └── presentation policy
          │
          ├── PlayOS Shell
          ├── Active Game
          ├── PlayOS Overlay
          └── Trusted System UI
```

The compositor is part of the PlayOS Runtime architecture, but remains a narrowly focused subsystem. Application launching, package execution, cgroups, crash supervision, suspend coordination, and permissions belong to runtime services rather than to the compositor.

## 3. Responsibilities

The compositor is responsible for:

- acquiring and retaining control of DRM/KMS outputs,
- selecting display modes, transforms, scaling, and refresh rates,
- presenting Wayland client buffers,
- managing fullscreen application surfaces,
- routing keyboard, pointer, touch, and controller-derived navigation input,
- switching visual focus between the shell and a running game,
- maintaining trusted overlay and system UI layers,
- reporting presentation and output state to runtime services,
- supporting direct scanout when safe and beneficial,
- restoring the shell when a game exits, crashes, or is terminated,
- supporting XWayland where compatibility requirements justify it.

The compositor is not responsible for package management, game discovery, accounts, entitlements, achievements, downloads, or save data.

## 4. Client and surface model

```text
System layer
    ├── emergency UI
    ├── crash UI
    └── power and security prompts

Overlay layer
    ├── quick menu
    ├── volume and brightness indicators
    ├── notifications and achievements
    ├── performance HUD
    └── virtual keyboard

Game layer
    └── one active game

Background layer
    └── PlayOS Shell
```

The shell remains alive while a game is running. When the game exits, the compositor reveals the existing shell immediately.

## 5. Surface roles

Early implementations may use `xdg-shell` and `wlr-layer-shell`, with runtime IPC providing temporary role association.

The long-term design should use private protocols:

```text
playos_shell_v1
playos_game_v1
playos_overlay_v1
playos_system_ui_v1
playos_compositor_control_v1
```

## 6. Relationship with the runtime

```text
playos-sessiond / runtime
    ├── launches the shell and games
    ├── owns process groups and cgroups
    ├── applies permissions
    ├── detects exits and crashes
    ├── coordinates suspend and resume
    └── requests presentation transitions

playos-compositor
    ├── activates shell or game surfaces
    ├── orders trusted overlays
    ├── routes input focus
    ├── manages outputs
    └── reports presentation state
```

Candidate control operations:

```text
ActivateShell()
ActivateGame(application_id, process_id)
ShowOverlay(overlay_id)
HideOverlay(overlay_id)
SetDisplayMode(output, mode)
SetRefreshRate(output, rate)
SetScalingMode(mode)
GetPresentationState()
```

## 7. Why wlroots and TinyWL

wlroots supplies mature implementations for DRM/KMS, session handling, libinput, output hotplug, buffer allocation, rendering integration, Wayland protocols, and XWayland.

TinyWL is useful for bring-up, but PlayOS should progressively replace its example policy with explicit PlayOS modules, tests, and protocols.

Borrow selectively from:

- **Cage** for fullscreen-first policy,
- **Gamescope** for gaming presentation concepts,
- **Sway** for mature wlroots integration,
- **Weston** for testing discipline.

## 8. Repository ownership

The compositor currently belongs in [`playos-runtime`](https://github.com/PlayOS-Foundation/playos-runtime), where it already exists under `compositor/`.

A separate repository should be created only when the compositor has an independent release cadence, stable private protocols, dedicated CI, a hardware test matrix, and a versioned IPC contract.

## 9. Initial protocol scope

```text
wl_compositor
wl_shm
wl_seat
wl_output
xdg_wm_base
linux_dmabuf
wp_presentation
zwlr_layer_shell_v1
```

Deferred initially:

- arbitrary workspaces,
- server-side decorations,
- drag and drop,
- general clipboard integration,
- remote desktop,
- unrestricted client overlays.

## 10. Input and focus policy

```text
Shell active
    → shell receives navigation and touch input

Game active
    → game receives gameplay input

Overlay active
    → trusted overlay receives navigation input
    → gameplay input may be suppressed or selectively forwarded

System UI active
    → trusted system UI has exclusive navigation focus
```

The Home button must remain available to the runtime even if a game is frozen.

## 11. Output policy

Initial requirements:

- ROG Ally internal-panel orientation,
- preferred mode selection,
- 60/120 Hz switching,
- logical scaling,
- touch-to-output mapping,
- external-display hotplug,
- safe fallback modes.

Later:

- VRR,
- HDR,
- color management,
- per-game output policy.

## 12. Presentation and performance

Measure:

- compositor startup time,
- time to first shell frame,
- shell/game transition time,
- input-to-photon latency,
- missed presentation deadlines,
- CPU/GPU usage,
- direct-scanout success,
- memory consumption.

## 13. Boot integration

```text
UEFI
  → Linux kernel and initramfs
  → GPU readiness
  → runtime prerequisites
  → playos-compositor
  → PlayOS Shell first frame
  → non-critical services continue asynchronously
```

Networking and marketplace services should not block the first shell frame.

## 14. Testing strategy

Automated:

- nested and headless startup,
- surface role registration,
- activation and focus transitions,
- overlay ordering,
- crash recovery,
- output hotplug,
- screenshot regression,
- protocol misuse.

Hardware:

- ROG Ally panel,
- 60/120 Hz switching,
- touch,
- controller reconnect,
- USB-C docks,
- suspend/resume,
- repeated launch and exit cycles.

## 15. Staged evolution

### Stage 1 — vertical slice

- DRM/KMS bring-up,
- fullscreen shell,
- keyboard/controller routing,
- one launched game,
- return to shell on exit.

### Stage 2 — runtime integration

- stable IPC,
- explicit roles,
- Home-button handling,
- pointer and touch,
- overlays,
- crash recovery,
- XWayland if required.

### Stage 3 — console services

- quick menu,
- virtual keyboard,
- performance HUD,
- brightness and volume overlays,
- suspend/resume,
- docked display handling.

### Stage 4 — measured optimizations

- direct scanout,
- presentation statistics,
- frame limiting,
- scaling,
- refresh-rate policy,
- VRR and HDR evaluation.

## 16. Non-goals

The compositor is not intended to:

- replace desktop compositors,
- provide a general Linux desktop,
- support unrestricted overlapping windows,
- own application lifecycle or package management,
- expose wlroots through the public PlayOS API,
- become independent of wlroots without measured justification.

## 17. Decision summary

```text
PlayOS Compositor
=
wlroots infrastructure
+ Cage-sized fullscreen policy
+ Gamescope-inspired gaming behaviour
+ Weston-style testing
+ PlayOS-specific roles and lifecycle integration
```

The runtime supervises applications, the compositor owns presentation and focus, and the shell remains a replaceable client.

See also:

- [Boot Model](03-boot-model.md)
- [Application Lifecycle](06-application-lifecycle.md)
- [Runtime IPC](09-runtime-ipc.md)
- [Input Routing](10-input-routing.md)
- [Game Switching](11-game-switching.md)
- [Overlay Integration](12-overlay-integration.md)
- [Reference Shell](../09-shell-and-ux/16-reference-shell.md)
