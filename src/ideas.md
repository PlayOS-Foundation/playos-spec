# PlayOS Architecture and Implementation Plan — Version 2.4

## Document Purpose

This document defines the product architecture, runtime model, platform boundaries, build system, and implementation roadmap for PlayOS.

PlayOS is not intended to be a conventional Linux distribution. It is a console operating environment that uses the Linux kernel as an invisible hardware-enablement layer and presents a controller-first PlayOS experience from power-on to shutdown.

The initial reference device is the ROG Ally. The first supported graphics stack is AMDGPU with Mesa. PlayOS standardizes on musl and deliberately excludes desktop Linux components that are not required by the console experience.

Version 2.4 formalizes the existing `playos-platform-api` repository as the owner of the public, engine-agnostic `libplayos` API and C ABI. It narrows `playos-runtime` to internal lifecycle transport, launch and control IPC, private protocol definitions, and operating-system integration. It also requires compositor implementation code that currently exists under `playos-runtime` to migrate into the dedicated `playos-compositor` repository.

---

# Part I — Product Definition

## 1. Executive Summary

PlayOS boots directly from UEFI into a small, immutable Linux system. A custom PID 1 process starts a purpose-built Wayland compositor based on wlroots. The compositor permanently owns DRM/KMS and launches the persistent PlayOS Shell as a trusted Wayland client. One game process may run at a time as another Wayland client.

The core runtime is:

```text
UEFI firmware
    |
    v
Linux EFI-stub kernel
    |
    v
Embedded initramfs
    |
    +-- playos-init            PID 1 and process supervisor
    +-- playos-compositor      wlroots compositor and display owner
    +-- playos-shell           persistent Raylib-powered Wayland client
    +-- libplayos              public C ABI from playos-platform-api
    +-- playos-runtime         internal IPC, lifecycle transport, OS integration
    +-- Mesa / Wayland / ALSA  platform libraries
    +-- GPU firmware
    |
    v
One active game process
```

The operating system is immutable. Games, saves, resources, logs, downloads, and updates live on a separate writable data partition.

The final user experience should be indistinguishable from a dedicated console:

```text
Power on
    -> PlayOS Shell
    -> Select game
    -> Game becomes foreground
    -> Press System button
    -> PlayOS UI appears immediately
    -> Resume or quit game
```

The recommended technical baseline is:

```text
Upstream Linux LTS
Buildroot with a PlayOS br2-external tree
musl libc only
Linux EFI stub
embedded initramfs
custom playos-init
custom playos-compositor built on wlroots
Wayland
Raylib-powered playos-shell
playos-platform-api providing the public libplayos C ABI
playos-runtime providing internal IPC and lifecycle transport
DRM/KMS + GBM + EGL + OpenGL ES
Mesa RadeonSI on AMD
ALSA
Linux evdev/HID
ext4 data partition
signed A/B system updates later
```

## 2. Product Goals

PlayOS should:

1. Boot directly into a controller-first shell with no visible Linux desktop or login screen.
2. Keep the operating system small, immutable, reproducible, and recoverable.
3. Use mature Linux drivers for GPU, audio, input, storage, power, and thermal management.
4. Keep `playos-shell` alive throughout the session so returning from a game is immediate.
5. Run one isolated game process at a time.
6. Recover cleanly when a game crashes or becomes unresponsive.
7. Allow trusted PlayOS overlays to appear above a running game.
8. Reserve system controls that games cannot consume directly.
9. Provide a stable, engine-agnostic PlayOS Platform API with an authoritative C ABI, plus Raylib and C++ convenience layers.
10. Store games and user state separately from the system image.
11. Support the ROG Ally first and add Intel graphics only after the AMD implementation is stable.
12. Deliver a polished console experience rather than a general-purpose Linux environment.

## 3. First-Release Non-Goals

The first release does not require:

- A desktop environment.
- X11 or Xwayland.
- systemd.
- A display or login manager.
- Multiple interactive users.
- Containers.
- A conventional package manager.
- A browser-based shell.
- A custom Linux kernel written from scratch.
- A custom GPU driver or OpenGL implementation.
- Multiple simultaneous games.
- Multi-GPU or hybrid-graphics support.
- Wi-Fi, Bluetooth, SSH, cloud saves, or a store during early bring-up.
- Full suspend/resume during the first boot milestone.
- HDR, VRR, recording, or streaming during the MVP.
- Any libc baseline other than musl.

## 4. Architectural Principles

### 4.1 Linux is the hardware layer, not the product

Linux provides:

- CPU scheduling and virtual memory.
- Process isolation.
- EFI, ACPI, PCIe, and IOMMU support.
- AMDGPU and other device drivers.
- DRM/KMS.
- USB, HID, and evdev.
- ALSA audio.
- NVMe and filesystems.
- Battery, thermal, and power-management interfaces.

PlayOS provides:

- Boot policy.
- Console UI and navigation.
- Display and foreground policy.
- Game launching and supervision.
- Game lifecycle events.
- Reserved system controls.
- Storage and save-data conventions.
- Updates and recovery.
- Public game APIs.

Games should target PlayOS APIs rather than depend directly on Linux implementation details wherever practical.

### 4.2 The system image is immutable

The kernel, initramfs, compositor, shell, Platform API library, internal runtime libraries, firmware, and core assets form a versioned system image. Runtime writes go to the data partition.

This provides:

- Predictable boot behavior.
- Safe rollback.
- Easy factory reset.
- Fewer corruption paths.
- Reproducible releases.
- Independent game and system updates.

### 4.3 The process model stays minimal but protected

Games remain userspace processes. They are never linked into the kernel or compositor.

The minimum production model is:

```text
playos-init
    +-- playos-compositor
    |       +-- playos-shell
    |       +-- optional playos-overlay
    |
    +-- active-game
```

This gives games a clean address space and crash boundary while keeping the system extremely small.

### 4.4 The compositor owns display policy permanently

`playos-compositor` is the only process that owns DRM/KMS and display policy. The shell and games are Wayland clients.

This avoids shell-to-game DRM handoff and enables:

- Seamless transitions.
- Reliable crash recovery.
- Trusted overlays.
- Centralized input focus.
- Consistent orientation and output policy.
- Future direct scanout without changing the application model.

### 4.5 wlroots supplies mechanisms; PlayOS supplies console policy

wlroots provides reusable building blocks for DRM/KMS, Wayland, rendering, outputs, buffers, seats, and input.

PlayOS defines:

- Which client is trusted as the shell.
- Which surface is the active game.
- Which UI may appear above a game.
- Which controls are reserved by the system.
- How launch, pause, resume, exit, and crash transitions work.
- Which protocols are exposed to clients.

The guiding rule is:

> wlroots implements mechanisms; `playos-compositor` implements console policy; Raylib renders what the player sees.

---

# Part II — End-to-End Runtime Architecture

## 5. System Overview

```text
+-----------------------------------------------------------+
|                       UEFI Firmware                       |
+-----------------------------+-----------------------------+
                              |
                              v
+-----------------------------------------------------------+
|                  PlayOS EFI Boot Artifact                 |
|                                                           |
|  Linux EFI stub                                           |
|  Linux kernel                                             |
|  kernel command line                                      |
|  embedded initramfs                                       |
+-----------------------------+-----------------------------+
                              |
                              v
+-----------------------------------------------------------+
|                       Linux Kernel                        |
|                                                           |
|  EFI / ACPI / PCIe / IOMMU                               |
|  scheduler / memory / processes                           |
|  AMDGPU / DRM/KMS                                         |
|  USB / HID / evdev                                        |
|  ALSA                                                     |
|  NVMe / ext4 / FAT                                        |
|  battery / thermal / power                                |
+-----------------------------+-----------------------------+
                              |
                              v
+-----------------------------------------------------------+
|                       PlayOS Runtime                       |
|                                                           |
|  playos-init                                              |
|  playos-compositor + wlroots                              |
|  playos-shell                                             |
|  optional playos-overlay                                  |
|  libplayos from playos-platform-api                       |
|  playos-runtime internal transports and service clients   |
|  Wayland / Mesa / ALSA libraries                          |
+-----------------------------+-----------------------------+
                              |
                              v
+-----------------------------------------------------------+
|                     Persistent Data                       |
|                                                           |
|  games / saves / resources / cache / logs / updates       |
+-----------------------------------------------------------+
```

## 6. Boot Architecture

### 6.1 Boot artifact

The first production-oriented design uses a UEFI-loadable artifact at:

```text
/EFI/BOOT/BOOTX64.EFI
```

The artifact contains, directly or as a unified image:

- Linux EFI stub.
- Linux kernel.
- Kernel command line.
- Embedded initramfs.

During development, the kernel and initramfs may also be produced separately for faster QEMU iteration. The acceptance path must always test real UEFI boot through OVMF or physical firmware.

### 6.2 Boot sequence

1. UEFI loads `BOOTX64.EFI`.
2. The EFI stub transfers control to the Linux kernel.
3. Linux initializes memory, interrupts, ACPI, PCIe, storage, input, graphics, and audio drivers.
4. Linux unpacks the initramfs into RAM.
5. Linux starts `/init`, implemented by `playos-init`, as PID 1.
6. `playos-init` mounts `/dev`, `/proc`, `/sys`, and `/run`.
7. `playos-init` discovers and mounts the PlayOS data partition.
8. `playos-init` starts `playos-compositor`.
9. The compositor initializes its wlroots backend, DRM/KMS, renderer, input seat, and Wayland socket.
10. The compositor launches `playos-shell` with the correct Wayland environment and trusted identity.
11. The shell maps its main fullscreen surface and displays the game library.
12. The user selects a game.
13. The shell sends a launch request to `playos-init` through restricted `playos-runtime` control IPC.
14. `playos-init` validates and spawns the game.
15. The game connects to Wayland and commits its first usable frame.
16. The compositor verifies the launch identity and makes the game foreground.
17. The application receives lifecycle and safe platform services through `playos-platform-api` while `playos-runtime` transports trusted internal events.
18. When the game exits or crashes, the compositor reveals and refocuses the existing shell surface.

### 6.3 PID 1 responsibilities

`playos-init` owns process and boot supervision. It should remain small and deterministic.

It is responsible for:

- Mounting virtual filesystems.
- Initializing runtime directories and logging.
- Discovering and mounting persistent storage.
- Starting and supervising `playos-compositor`.
- Validating game manifests and launch requests.
- Spawning, monitoring, terminating, and reaping game processes.
- Creating game process groups and lifecycle channels.
- Handling reboot, shutdown, and recovery requests.
- Restarting the compositor after a recoverable failure.
- Entering recovery mode after repeated boot or compositor failure.

It should not contain UI, rendering, network policy, or game-specific logic.

## 7. Runtime Components and Responsibilities

### 7.1 `playos-init`

Owns:

- Boot and service lifecycle.
- Process creation and supervision.
- Game launch validation.
- Exit status and crash reporting.
- Forced pause or termination fallback.
- Shutdown, reboot, and recovery.

It does not own surfaces, focus, or rendering.

### 7.2 `playos-compositor`

Owns:

- DRM/KMS devices and output state.
- The wlroots backend, renderer, allocator, and scene.
- The Wayland display and socket.
- Display selection, orientation, refresh rate, and hotplug policy.
- Surface roles, z-order, visibility, and focus.
- Trusted shell and overlay identity.
- Expected game identity and first-frame activation.
- Reserved system controls.
- Input routing between shell, game, and overlay.
- Direct-scanout eligibility and composition fallback.
- Returning to the shell after game exit or failure.

It does not install games, manage saves, or act as a general process supervisor.

### 7.3 `playos-shell`

Owns:

- The persistent console UI.
- Controller-first navigation.
- Game discovery and metadata presentation.
- User-facing launch, resume, quit, and crash flows.
- Settings and status screens.
- Initiating launch requests to `playos-init`.
- Rendering the main shell and trusted PlayOS UI with Raylib.
- Preserving UI state while a game is foreground.

The shell remains alive while a game runs. It stops or heavily throttles its main fullscreen rendering but remains available to render system UI.

### 7.4 `playos-overlay`

An overlay may initially be a separate trusted Raylib Wayland client because stock Raylib is most comfortable with one native surface per process.

It owns only overlay presentation, such as:

- Quick menu.
- Volume and brightness indicators.
- Power menu.
- Notifications.
- Virtual keyboard.

It may later be merged into a multi-surface `playos-shell` backend.

### 7.5 Active game process

The game owns:

- Its own address space and Raylib state.
- Its Wayland surface and rendering context.
- Its audio streams.
- Its assigned input stream.
- Its save and cache directories.

The game may not:

- Become DRM master.
- Reconfigure displays directly.
- Mount or format filesystems.
- Modify the immutable system image.
- Access another game's private data.
- Consume reserved PlayOS system actions.

### 7.6 `playos-platform-api` and `libplayos`

`playos-platform-api` owns the public, engine-agnostic PlayOS application contract. Its primary binary artifact is `libplayos`, and its authoritative compatibility boundary is a stable C ABI.

It exposes:

- Lifecycle events.
- Assigned install, save, and cache paths.
- Device and capability information.
- Logical input conventions.
- Logging.
- Audio, storage, display, and power queries that are safe for applications.
- Narrow requests for approved system actions.

The repository may also provide C++ wrappers and engine adapters, including a Raylib integration layer, but those layers must be implemented above the C ABI rather than replace it. Games, samples, and ordinary shell code use `playos-platform-api`; they do not consume compositor internals or privileged control protocols directly.

### 7.7 `playos-runtime`

`playos-runtime` owns internal PlayOS integration mechanisms rather than the public application API. It contains:

- Versioned launch and control IPC definitions.
- Lifecycle-event transport.
- Private Wayland protocol XML shared with the compositor and trusted clients.
- Restricted client libraries used by trusted system components.
- Process, session, and operating-system integration helpers.
- The PlayOS backend transport used by `playos-platform-api`.

Privileged actions cross these narrow internal interfaces. A normal game reaches them only through the validated public Platform API surface. `playos-runtime` must not own DRM/KMS policy or the compositor implementation.

## 8. Console Lifecycle

### 8.1 Game launch

The launch responsibility is intentionally split:

```text
playos-shell         chooses and requests the game through restricted control IPC
playos-init          validates, spawns, supervises, and terminates it
playos-compositor    identifies its surface and controls presentation
playos-runtime       transports lifecycle and control messages internally
playos-platform-api  exposes lifecycle events and safe services to the application
```

Detailed launch flow:

1. The player selects a game in the shell.
2. The shell sends `LaunchGame(game_id)` over PlayOS control IPC.
3. `playos-init` validates the manifest, executable, permissions, and one-game rule.
4. `playos-init` prepares save paths, cache paths, process group, lifecycle channel, and a one-time launch identity.
5. The shell shows a Launching state and remains visible.
6. `playos-init` spawns the game with `WAYLAND_DISPLAY` and PlayOS environment variables.
7. The game connects to the compositor and creates its surface.
8. The compositor matches the client to the expected launch identity.
9. The compositor waits for the first valid committed buffer.
10. Only then does it switch foreground from shell to game.
11. The shell remains alive and background-throttled.

This first-frame rule avoids switching to a black screen while the game is still initializing.

### 8.2 System button and backgrounding

PlayOS reserves a logical system action:

```text
PLAYOS_BUTTON_SYSTEM
```

Each hardware profile maps a physical key to this action. The action never reaches the game directly.

When pressed during gameplay, the compositor:

1. Removes normal input focus from the game.
2. Sends a background lifecycle event.
3. Shows the shell or trusted overlay.
4. Routes UI input to PlayOS.

PlayOS-native games receive cooperative events:

```c
typedef enum PlayOSLifecycleEvent {
    PLAYOS_LIFECYCLE_FOREGROUND,
    PLAYOS_LIFECYCLE_BACKGROUND,
    PLAYOS_LIFECYCLE_SUSPEND,
    PLAYOS_LIFECYCLE_RESUME,
    PLAYOS_LIFECYCLE_TERMINATE
} PlayOSLifecycleEvent;
```

The lifecycle enum and polling API are defined by `playos-platform-api`; `playos-runtime` transports the events from trusted system components. A cooperative game should pause gameplay, stop normal input processing, lower or mute audio, and reduce rendering while backgrounded.

For non-cooperative games, `playos-init` may use `SIGSTOP` and `SIGCONT` as a compatibility fallback. The compositor requests that action; it does not manipulate Unix processes itself.

### 8.3 Overlay presentation

The scene order is:

```text
1. active game surface
2. optional dimming layer
3. trusted PlayOS overlay surface
4. notifications and cursor, when enabled
```

The overlay is a separate Wayland surface because one `wl_surface` may have only one role.

Overlay flow:

```text
System button
    -> compositor intercepts action
    -> game loses focus
    -> overlay maps above game
    -> overlay receives UI input
    -> game is cooperatively or forcibly paused
```

Closing the overlay reverses the flow and returns focus to the game.

### 8.4 Game exit and crash recovery

When the game exits:

1. `playos-init` records the exit status.
2. The compositor destroys or ignores stale game surfaces.
3. The compositor reveals the already-running shell surface.
4. Focus returns to the shell.
5. The shell restores the previous library position and shows any required message.

A game crash must never reveal a terminal or leave the display black.

### 8.5 Compositor state machine

```text
SHELL_FOREGROUND
    |
    | launch accepted
    v
GAME_STARTING
    |
    | first valid game frame
    v
GAME_FOREGROUND
    |
    | System button
    v
PLAYOS_UI_FOREGROUND_WITH_GAME_BACKGROUND
    |
    +-- Resume -> GAME_FOREGROUND
    |
    +-- Quit -> TERMINATING_GAME -> SHELL_FOREGROUND

GAME_FOREGROUND
    |
    | game exits or crashes
    v
SHELL_FOREGROUND
```

This state machine is a central PlayOS contract and should be explicitly tested.

## 9. Wayland and wlroots Boundary

### 9.1 What wlroots already provides

wlroots supplies:

- DRM/KMS, nested, headless, and input backends.
- Renderer and allocator abstractions.
- Output, buffer, and scene primitives.
- Core Wayland compositor building blocks.
- XDG shell support.
- Seat, keyboard, pointer, touch, and libinput integration.
- Surface commit, map, unmap, destroy, and frame events.

### 9.2 What PlayOS must implement

`playos-compositor` must implement:

- Trusted shell identity.
- Expected game launch identity.
- One foreground experience at a time.
- Fullscreen shell and game policy.
- First-frame launch switching.
- Reserved system controls.
- Overlay authorization and stacking.
- Focus and input routing.
- Output and orientation policy.
- Direct-scanout policy.
- Lifecycle state transitions.
- Crash recovery.
- Protocol permissions and client restrictions.

### 9.3 Private PlayOS Wayland protocol

Standard Wayland protocols should be used wherever possible. A small private protocol should cover only console presentation and lifecycle concerns.

Initial capabilities may include:

- Registering the trusted shell.
- Registering an expected game launch identity.
- Assigning shell, game, and trusted overlay roles.
- Reporting surface readiness.
- Reporting foreground and background transitions.
- Requesting return to shell or resume game.
- Reporting output size, scale, refresh rate, and orientation.

The protocol must be versioned and generated with `wayland-scanner`.

The following do not belong in the Wayland protocol:

- Game installation.
- Save management.
- Networking.
- Updates.
- Account services.
- General process management.

Those use PlayOS control or service IPC.

---
# Part III — Platform Subsystems

## 10. Graphics Architecture

### 10.1 Initial graphics stack

```text
Raylib shell or game
    |
    v
Raylib Wayland / PlayOS backend
    |
    v
Wayland protocol
    |
    v
playos-compositor + wlroots
    |
    v
GBM / EGL / OpenGL ES renderer
    |
    v
DRM/KMS + AMDGPU
    |
    v
Display
```

The first release uses:

- DRM/KMS for display control.
- wlroots for compositor infrastructure.
- Wayland for shell and game presentation.
- GBM and EGL for buffer and context integration.
- OpenGL ES 3.0 or 3.1 for Raylib clients.
- Mesa RadeonSI on AMD.

Vulkan may be introduced later without changing the compositor ownership model.

### 10.2 Raylib integration

The shell and games should use a dedicated Raylib PlayOS backend, for example:

```text
src/platforms/rcore_playos.c
```

The backend should adapt Raylib's Wayland support and integrate:

- Fullscreen Wayland surfaces.
- EGL context creation.
- Frame callbacks and presentation timing.
- PlayOS lifecycle events.
- Trusted client identity where applicable.
- Logical input mapping.
- Exit and foreground requests.
- PlayOS save and cache paths.

Initially unsupported desktop features may include:

- Arbitrary window positioning.
- Multiple desktop windows.
- Decorations.
- File drag and drop.
- Clipboard integration.
- User-driven resize.

### 10.3 Direct scanout

When a game fully covers the output and no overlay is visible, the compositor should attempt direct scanout if the client buffer is compatible with the output.

```text
Compatible fullscreen game buffer
    -> DRM plane
    -> display
```

When an overlay appears, or when format, scaling, transform, or synchronization prevents direct scanout, the compositor falls back to normal composition.

Direct scanout is an optimization, not a correctness requirement for the MVP.

### 10.4 GPU discovery

Do not assume `/dev/dri/card0` is always the intended device.

The compositor should:

1. Enumerate DRM devices and render nodes.
2. Resolve each device to its PCI identity.
3. Identify the device connected to the active display.
4. Validate renderer initialization.
5. Select the scanout and render device.
6. Fall back to a diagnostic or software path if initialization fails.

Initial supported vendor IDs:

```text
AMD       0x1002
Intel     0x8086
```

### 10.5 AMD implementation

Kernel:

- `amdgpu`.
- AMD display core.
- Required ROG Ally firmware.

Userspace:

- Mesa RadeonSI for OpenGL ES/OpenGL.
- libdrm.
- GBM.
- EGL.

RADV may be added when Vulkan becomes a product requirement.

### 10.6 Intel expansion

After the AMD baseline is stable, Intel support may use:

Kernel:

- `i915` or `xe`, depending on supported hardware.
- Required firmware.

Userspace:

- Mesa Iris.
- libdrm.
- GBM.
- EGL.

ANV may be added with Vulkan support later.

### 10.7 Recovery graphics

Recovery mode must remain usable when accelerated graphics fails.

Possible fallback paths:

- SimpleDRM or firmware framebuffer.
- Text or software-rendered diagnostic UI.
- A minimal recovery menu with storage, log, rollback, and reboot actions.

## 11. Input Architecture

### 11.1 Input path

```text
Controller / keyboard / touch
    |
    v
Linux HID and device drivers
    |
    v
evdev / libinput
    |
    v
playos-compositor
    |
    +-- reserved actions -> PlayOS only
    +-- overlay visible -> overlay
    +-- game foreground -> game
    +-- otherwise -> shell
```

The compositor owns focus and reserved input policy during Phase 1.

A later `playos-input` service may centralize remapping, profiles, virtual controllers, gyro, haptics, and multi-controller assignment.

### 11.2 Logical input model

Games should receive logical controls rather than raw device codes:

```text
PLAYOS_BUTTON_SOUTH
PLAYOS_BUTTON_EAST
PLAYOS_BUTTON_WEST
PLAYOS_BUTTON_NORTH
PLAYOS_BUTTON_START
PLAYOS_BUTTON_SELECT
PLAYOS_BUTTON_SYSTEM
PLAYOS_BUTTON_QUICK_MENU
PLAYOS_AXIS_LEFT_X
PLAYOS_AXIS_LEFT_Y
PLAYOS_AXIS_RIGHT_X
PLAYOS_AXIS_RIGHT_Y
PLAYOS_AXIS_LEFT_TRIGGER
PLAYOS_AXIS_RIGHT_TRIGGER
```

`PLAYOS_BUTTON_SYSTEM` is reserved and is not delivered to games.

### 11.3 Initial ROG Ally scope

Support first:

- D-pad.
- A/B/X/Y.
- Both analog sticks.
- Both triggers.
- Menu and View.
- One development fallback for the System action.

Defer until later:

- Rear buttons.
- Vendor-specific command buttons.
- Gyroscope.
- Haptics.
- RGB control.
- Controller mode switching.

## 12. Audio Architecture

### 12.1 Initial path

```text
Raylib audio
    |
    v
PlayOS audio backend
    |
    v
ALSA PCM
    |
    v
Kernel ALSA driver
    |
    v
Speakers or headphones
```

No PulseAudio or PipeWire is required for the MVP.

### 12.2 Initial policy

Support:

- Stereo PCM output.
- Master volume.
- Mute.
- Built-in speaker and headphone selection.
- Device-loss and underrun recovery.

The first implementation may allow one foreground audio owner. When the game becomes foreground, shell audio stops. When the game backgrounds or exits, shell audio resumes.

A dedicated PlayOS audio service is introduced only when simultaneous shell and game audio, notifications over games, Bluetooth audio, or per-application mixing becomes necessary.

## 13. Storage Architecture

### 13.1 Partition model

Prototype layout:

```text
GPT disk

Partition 1: EFI System Partition
    FAT32
    PlayOS boot artifact

Partition 2: PlayOS Data
    ext4
    games and writable state
```

Production A/B layout:

```text
Partition 1: EFI System Partition
Partition 2: PlayOS system A
Partition 3: PlayOS system B
Partition 4: PlayOS data
```

The exact system-slot representation may remain EFI-image based, partition based, or a combination. The invariant is that writable game and user data is separate from replaceable system state.

### 13.2 Persistent directory layout

Mount the writable partition at `/data`:

```text
/data/
    games/
    saves/
    profiles/
    resources/
    cache/
    downloads/
    logs/
    updates/
    screenshots/
    config/
```

Per-game layout:

```text
/data/games/<game-id>/
    manifest.json
    bin/
    assets/
    shaders/
    licenses/

/data/saves/<game-id>/
    profiles/
    autosaves/
    settings/

/data/cache/<game-id>/
    shaders/
    compiled-assets/
    temporary/
```

### 13.3 Filesystem choice

Use ext4 initially because it is mature, journaled, well supported, and easy to recover.

Do not design a custom filesystem for the first release. A PlayOS package or virtual filesystem can be layered above ext4 later.

### 13.4 First-boot provisioning

PlayOS must never silently format an unknown disk.

Provisioning should:

1. Search for the expected partition GUID, label, or UUID.
2. Mount it if valid.
3. Enter provisioning mode if absent.
4. Show the target and destructive impact.
5. Require explicit confirmation or a manufacturing flag.
6. Create the filesystem and expected metadata.
7. Create the directory tree.
8. Write a storage-version marker.

### 13.5 Factory reset

A normal factory reset operates on writable state and leaves the immutable boot system intact.

The user may choose whether to erase:

- Configuration.
- Installed games.
- Saves.
- Cache.
- Downloads.
- Logs.

## 14. Game Packaging and Platform API

### 14.1 Initial game package

A game may initially be an ordinary directory:

```text
<game-id>/
    manifest.json
    bin/game
    assets/
    licenses/
```

Example manifest:

```json
{
  "id": "com.example.game",
  "name": "Example Game",
  "version": "1.0.0",
  "executable": "bin/game",
  "api_version": 1,
  "graphics": "gles3",
  "architecture": "x86_64",
  "controllers": true,
  "network": false
}
```

### 14.2 Later `.play` package

A future package may be a signed archive, SquashFS image, or content-addressed bundle.

Required properties:

- Deterministic metadata.
- Signature verification.
- Integrity hashes.
- Atomic installation.
- Versioned migrations.
- Strict save-data separation.

### 14.3 PlayOS Platform API goals

The public API is specified in `playos-spec` and implemented by `playos-platform-api`. It should hide Linux implementation details and expose stable capabilities through an authoritative C ABI. The same API surface should remain usable from Raylib, SDL, Godot, custom engines, and non-Linux SDK targets through replaceable backends.

Initial groups:

```text
playos_system
playos_display
playos_input
playos_audio
playos_storage
playos_game
playos_power
playos_logging
```

Examples:

System:

- API and OS version.
- Device model.
- CPU and GPU information.
- Memory and locale.

Display:

- Resolution and refresh rate.
- Orientation and scale.
- V-sync preference.

Input:

- Logical controller state.
- Device connection events.
- Lifecycle-safe input state.

Storage:

- Install, save, and cache directories.
- Free-space query.
- Atomic replacement helpers.

Power:

- Battery and charging state.
- Thermal state.
- Approved performance-profile request.

Logging:

- Structured logs.
- Session identifiers.
- Crash markers.

### 14.4 API delivery and internal transport

`playos-platform-api` initially delivers:

- Public C headers under `include/playos/`.
- `libplayos.so` for PlayOS runtime devices.
- A static SDK library where appropriate for SDK-only targets.
- Optional C++ wrappers that preserve the C ABI as the source of truth.
- Engine adapters, including the Raylib PlayOS layer.

The PlayOS backend may consume launch-time environment values and a lifecycle or service channel implemented by `playos-runtime`. Internal transport details, privileged launch control, compositor control, and private Wayland protocols are not public Platform API contracts.

The C ABI must be consumable from C, C++, Rust, Zig, and other languages. Public ABI changes require versioning and compatibility review in `playos-spec`.

## 15. Security Model

### 15.1 Initial development trust

Early bring-up images may run with broader privileges to accelerate hardware debugging. That is not the production model.

### 15.2 Production direction

- `playos-init` runs as root.
- `playos-compositor` receives only required display and input privileges.
- `playos-shell` runs as a dedicated service user where practical.
- Games run as an unprivileged game identity.
- Games receive per-title save and cache directories.
- Games link to `playos-platform-api`; privileged `playos-runtime` control interfaces are restricted to trusted components.
- The system image is read-only.
- Remote development services are absent from retail builds.

### 15.3 Game restrictions

A normal game should not be able to:

- Modify the system image.
- Access another game's saves.
- Open DRM primary nodes.
- Mount filesystems.
- Change kernel modules.
- Format storage.
- Invoke unrestricted shutdown, update, or power controls.
- Create trusted overlays.
- Synthesize reserved system input.
- Connect to privileged launch, compositor-control, or trusted-role IPC endpoints.

Later hardening may use capabilities, seccomp, Landlock, namespaces where useful, signed manifests, and package integrity verification.

### 15.4 Secure Boot

Secure Boot is a later production requirement. The signed chain should cover:

- PlayOS EFI artifact.
- Kernel and embedded initramfs.
- External kernel modules, if any.
- A/B update metadata.

Development builds may initially run with Secure Boot disabled.

## 16. Updates, Recovery, and Failure Handling

### 16.1 A/B updates

The update flow should:

1. Download a signed system image.
2. Verify its signature and compatibility.
3. Write the inactive slot.
4. Mark it as the next boot candidate.
5. Boot it once.
6. Record a health-success marker.
7. Roll back automatically after repeated failure.

Games and saves remain untouched.

RAUC is the preferred first update engine unless a simpler EFI-image-specific updater proves sufficient.

### 16.2 Recovery mode

Recovery must work without accelerated graphics and should provide:

- System-slot selection.
- Rollback.
- Filesystem check.
- Log export.
- Factory reset.
- Reboot and shutdown.

### 16.3 Failure policy

Game failure:

- Return to shell.
- Record exit status and crash metadata.
- Offer restart or report options.

Shell failure:

- Restart the shell client without restarting the compositor where possible.

Compositor failure:

- `playos-init` restarts the graphical session.
- Repeated failure enters recovery.

Kernel or boot failure:

- Boot counting and A/B rollback select the previous known-good image.

---
# Part IV — Build, Kernel, and Development Model

## 17. Build System

### 17.1 Repository ownership and dependency direction

The MVP implementation uses six repositories:

| Repository | Ownership |
|---|---|
| `playos-spec` | Architecture, public contracts, RFCs, ADRs, schemas, roadmap, and product documentation. |
| `playos-platform-api` | Public `libplayos` C ABI, portable implementations, C++ wrappers, and engine adapters. |
| `playos-runtime` | Internal lifecycle transport, launch and control IPC, private Wayland protocols, restricted service clients, and OS integration. |
| `playos-compositor` | wlroots compositor, DRM/KMS ownership, surfaces, focus, trusted roles, overlays, and reserved-input policy. |
| `playos-shell` | Raylib controller-first shell and trusted user experience. |
| `playos-refdistro` | Buildroot integration, kernel and initramfs configuration, `playos-init`, packaging, image assembly, installer, recovery, and release pinning. |

Dependency direction:

```text
playos-spec
    defines contracts for every implementation repository

playos-runtime
    owns private transports and protocols
       ↑                         ↑
       │                         │
playos-platform-api       playos-compositor
       ↑
       │
playos-shell and games

playos-shell may additionally use a restricted playos-runtime control client
for trusted launch and system requests.

playos-refdistro pins, packages, and assembles all runtime components.
```

Rules:

- Public application headers and ABI belong only to `playos-platform-api`.
- Private launch, lifecycle-transport, compositor-control, and Wayland protocol definitions belong to `playos-runtime`.
- DRM/KMS, wlroots, surface, focus, and input-routing implementation belongs to `playos-compositor`.
- `playos-refdistro` may package and pin components but must not redefine their public contracts.
- Existing compositor code under `playos-runtime` must migrate to `playos-compositor`; after migration there must be one active compositor implementation, not two divergent copies.
- The older Alpine-oriented refdistro map is superseded for this architecture baseline by the Buildroot model in this document.

### 17.2 Buildroot strategy

Use the official Buildroot repository as a pinned Git submodule. Do not fork it initially.

All PlayOS-specific work lives in a `br2-external` tree:

```text
playos-refdistro/
    buildroot/                  official pinned submodule
    br2-external/
        external.desc
        Config.in
        external.mk
        configs/
            playos_qemu_x86_64_defconfig
            playos_rog_ally_defconfig
            playos_intel_pc_defconfig       later
        board/playos/
            common/
            qemu-x86_64/
            rog-ally/
            intel-pc/                       later
        package/
            playos-init/
            playos-platform-api/
            playos-runtime/
            playos-compositor/
            playos-shell/
            playos-overlay/
            raylib-playos/
        patches/
            linux/
            wlroots/
            raylib/
    src/
    protocols/
    scripts/
    docs/
    .github/workflows/
    Makefile
    versions.lock
```

`versions.lock` must pin full commits for `playos-platform-api`, `playos-runtime`, `playos-compositor`, and `playos-shell`, in addition to Buildroot, Linux, toolchain, configuration, and patch-set inputs.

A Buildroot fork is justified only when a required change cannot be represented as an external package, board configuration, patch, rootfs overlay, post-build script, or post-image script.

### 17.3 Toolchain

Use:

```text
GCC initially
musl libc only
GNU binutils or lld as supported by the selected Buildroot configuration
CMake or Meson per component
wayland-scanner for private protocols
```

Clang builds may be added to CI for diagnostics and sanitizers, but GCC remains the first reference compiler.

### 17.4 Build variants

Development image:

- BusyBox or a minimal diagnostic shell.
- Serial logs.
- SSH only after networking is introduced.
- `strace`, `gdbserver`, `evtest`, `modetest`, and graphics diagnostics as needed.
- Extra assertions and debug symbols.

Production image:

- No interactive shell in normal mode.
- No remote debug service.
- Minimal utilities.
- Signed artifacts.
- Bounded persistent logs.
- Recovery entry point.

### 17.5 Boot image strategy

Produce both:

Fast development outputs:

```text
bzImage
rootfs.cpio.zst
```

UEFI acceptance output:

```text
playos-esp.img
    /EFI/BOOT/BOOTX64.EFI
```

The acceptance image must boot through OVMF rather than QEMU's direct `-kernel` shortcut.

### 17.6 Stable developer commands

The repository should expose a small command surface:

```text
make setup
make qemu-config
make qemu-build
make qemu-run
make ally-config
make ally-build
make clean
```

Developers should not need to memorize raw Buildroot command lines.

## 18. Kernel Configuration Strategy

Use an upstream Linux LTS branch as the release base and keep a separate newer test track for hardware-enablement evaluation.

Suggested policy:

```text
playos-kernel-lts     release and qualification branch
playos-kernel-next    newer stable branch for hardware testing
```

Start from a known-working ROG Ally configuration and remove features gradually. Do not begin from an aggressively minimal embedded configuration.

Required subsystem groups include:

- x86-64 and EFI stub.
- ACPI and PCIe.
- IOMMU.
- devtmpfs, procfs, sysfs, tmpfs, initramfs.
- serial and early console for diagnostics.
- DRM/KMS, SimpleDRM, and AMDGPU.
- USB xHCI.
- HID, evdev, and ASUS-specific input support where needed.
- ALSA HDA/SoC/ACP support required by the device.
- NVMe.
- FAT and ext4.
- thermal, battery, power-supply, and AMD P-state support.
- watchdog and recovery-relevant facilities.

Remove only after qualification:

- Unused server filesystems.
- Unused network protocols.
- Unused GPU and sound drivers.
- Legacy buses not present on supported hardware.
- Virtualization host features not needed by the product.
- Excessive debug features in retail builds.

Keep drivers modular during discovery when that improves iteration. Convert the final required hardware set to built-ins when the support matrix is stable.

## 19. Development, Testing, and Performance

### 19.1 QEMU and OVMF

Use QEMU with OVMF for:

- UEFI boot.
- initramfs validation.
- PID 1 behavior.
- compositor headless or nested testing.
- process lifecycle.
- storage provisioning on virtual disks.
- update and rollback logic.
- automated boot checks.

QEMU does not replace physical testing for AMDGPU, Ally input, audio, ACPI, suspend, battery, or thermal behavior.

### 19.2 Physical-device testing

Maintain at least one ROG Ally as a continuous hardware target.

Required smoke tests should cover:

- Cold boot.
- Repeated reboot.
- Shell rendering.
- Controller navigation.
- Game launch and exit.
- System-button transition.
- Overlay focus.
- Game crash recovery.
- Audio output.
- Data persistence.
- Battery and thermal reporting.

### 19.3 CI layers

```text
Layer 1: host unit tests
Layer 2: Buildroot clean build
Layer 3: QEMU/OVMF boot test
Layer 4: compositor and shell smoke test
Layer 5: game lifecycle integration test
Layer 6: physical ROG Ally smoke test
Layer 7: long-running stability and update tests
```

Useful tools include:

- Kernel selftests where relevant.
- IGT GPU tools for DRM/KMS.
- Piglit for OpenGL validation.
- apitrace for graphics investigation.
- `perf` for CPU profiling.
- AddressSanitizer and UndefinedBehaviorSanitizer in test builds.
- RenderDoc later where the path is supported.

### 19.4 Logging

Development logs should capture:

- Kernel boot and driver selection.
- DRM device, connector, mode, and renderer.
- Wayland client identity and surface lifecycle.
- Compositor state transitions.
- Direct-scanout attempts and fallback reasons.
- Game launch command and exit status.
- Input-device discovery and logical mappings.
- Audio-device selection and underruns.
- Storage mount and recovery events.

Production logging must be bounded, privacy-aware, and recoverable without exposing a normal shell.

### 19.5 Performance rules

Do not optimize process count merely for appearance. Sleeping processes do not meaningfully compete with the game.

Prioritize:

- Correct AMDGPU power and clock behavior.
- Direct scanout for eligible fullscreen games.
- Stable frame pacing.
- Avoiding unnecessary composition.
- Keeping hidden shell rendering stopped or throttled.
- Minimal allocations in the compositor frame loop.
- Fast first-frame switching.
- Reliable audio buffer sizing.
- Safe thermal limits.

Performance changes must be measured on physical hardware before becoming policy.

---
# Part V — Delivery Plan

## 20. MVP Definition

The first meaningful PlayOS MVP is complete when:

1. The ROG Ally boots directly from UEFI into PlayOS.
2. The Linux kernel and initramfs are available as a UEFI-bootable artifact.
3. `playos-init` runs as PID 1.
4. `playos-compositor` permanently owns DRM/KMS and the Wayland session.
5. `playos-shell` remains alive as the persistent controller-first UI.
6. The compositor uses wlroots with AMDGPU, DRM/KMS, GBM, EGL, and Mesa.
7. The shell renders through Wayland using the Raylib PlayOS backend.
8. The shell and sample game consume the public `playos-platform-api` C ABI.
9. Trusted launch, lifecycle transport, and compositor-control mechanisms remain internal to `playos-runtime`.
10. The shell requests a game launch and `playos-init` spawns and supervises it.
11. The compositor waits for the game's first valid frame before making it foreground.
12. The game renders with hardware acceleration and receives normal controller input.
13. The reserved System button returns to PlayOS UI and backgrounds or pauses the game.
14. Resume returns to the same running game without restarting it.
15. The game outputs audio through ALSA.
16. Clean exit and crash both return safely to the existing shell.
17. Games and saves persist on a separate ext4 partition.
18. The system image is immutable.
19. Recovery remains usable without accelerated graphics.

## 21. Implementation Sprints

The detailed delivery plan is maintained in standalone sprint documents. This keeps the architecture readable while allowing each sprint to evolve as an executable work package.

| Sprint | Focus | Primary outcome |
|---:|---|---|
| [0](Sprint-0.md) | Build and UEFI Foundation | A six-repository, reproducible Buildroot factory that boots a minimal PlayOS EFI image through QEMU/OVMF. |
| [1](Sprint-1.md) | `playos-init` and Minimal Boot Supervision | A real `playos-init` running as PID 1 with versioned private control IPC. |
| [2](Sprint-2.md) | Compositor Skeleton and Wayland Session | A minimal wlroots compositor that creates a Wayland session and presents one trusted fullscreen client. |
| [3](Sprint-3.md) | ROG Ally Kernel and Device Bring-Up | Reliable USB boot, essential devices, and the first qualified Platform API input backend contract. |
| [4](Sprint-4.md) | AMDGPU and Native DRM/KMS | The compositor permanently owns the Ally display through AMDGPU and DRM/KMS. |
| [5](Sprint-5.md) | Raylib-Powered PlayOS Shell | A hardware-accelerated Raylib shell consuming the public PlayOS Platform API. |
| [6](Sprint-6.md) | Persistent Storage and Game Discovery | Persistent ext4 storage, safe Platform API paths, and shell-visible game discovery. |
| [7](Sprint-7.md) | Game Launch, Lifecycle, System Button, and Overlay | The complete console lifecycle with a public application API and private trusted control path. |
| [8](Sprint-8.md) | ALSA Audio | Reliable ALSA audio with safe public controls across lifecycle transitions. |
| [9](Sprint-9.md) | Power, Battery, Thermal, and Suspend Foundations | Safe power behavior exposed through a restricted public Platform API. |
| [10](Sprint-10.md) | Installer and Internal-Disk Deployment | A tested installation path from removable media to the ROG Ally internal SSD. |
| [11](Sprint-11.md) | Immutable Images and A/B Updates | Signed, atomic A/B system updates with automatic rollback. |
| [12](Sprint-12.md) | Security Hardening | A hardened boundary between public Platform API calls, trusted runtime control, and games. |
| [13](Sprint-13.md) | Intel Expansion | Proof that the architecture and Platform API backend model are portable to Intel. |
| [14](Sprint-14.md) | Production Readiness | A signed preview release with a versioned public Platform API and separated trusted integration docs. |

Execution rules:

- Sprints follow dependency order unless an ADR explicitly changes the sequence.
- A sprint begins only after the required predecessor exit criteria are satisfied.
- Each sprint must end with a demonstrable and testable system outcome.
- Architecture changes discovered during implementation must be captured in `playos-spec` and reflected here.

## 22. Post-MVP Roadmap

Add only when the core console lifecycle is stable:

- `playos-device` for hardware and power policy.
- `playos-net` with iwd for Wi-Fi.
- Dropbear SSH in explicit Developer Mode only.
- `playos-update` as a PlayOS wrapper around the update engine.
- A dedicated `playos-input` service for remapping, virtual gamepads, gyro, and haptics.
- A dedicated audio service for mixing and notifications over games.
- Bluetooth.
- Fast and fully qualified suspend/resume.
- Rear-button and special-button support.
- Screenshots and recording.
- Vulkan.
- VRR and HDR.
- External-display profiles.
- Download manager and store integration.
- Cloud saves and user accounts.
- Multiple local profiles.
- Signed `.play` content packages.
- Delta updates.
- Telemetry only with explicit user consent.

## 23. Final Reference Baseline

```text
Primary device:
    ROG Ally

Firmware and boot:
    UEFI
    Linux EFI stub
    embedded initramfs

Kernel:
    upstream Linux LTS
    ROG Ally-specific configuration

Image generation:
    official pinned Buildroot submodule
    PlayOS br2-external tree

C library:
    musl only

PID 1:
    playos-init

Display server:
    playos-compositor built on wlroots

Public application API:
    playos-platform-api
    libplayos authoritative C ABI
    optional C++ and engine wrappers

Internal runtime integration:
    playos-runtime lifecycle transport
    launch and control IPC
    private Wayland protocols

Persistent UI:
    playos-shell as a trusted Raylib Wayland client

System UI during gameplay:
    trusted overlay surface or playos-overlay client

Game runtime:
    one isolated Wayland game process at a time

Graphics:
    DRM/KMS
    wlroots
    Wayland
    GBM / EGL / OpenGL ES
    AMDGPU + Mesa RadeonSI

Input:
    Linux HID / evdev / libinput
    compositor-enforced reserved system actions
    PlayOS logical mapping

Audio:
    ALSA directly for MVP

Storage:
    immutable system image
    separate ext4 data partition

Updates:
    signed A/B images

Later expansion:
    Intel graphics
    networking and developer SSH
    dedicated device, input, audio, and update services
```

The defining architectural principle is:

> PlayOS is a console operating environment built on a minimal Linux hardware layer. `playos-init` owns processes, `playos-compositor` owns display and focus, `playos-shell` owns the user experience, `playos-platform-api` owns the public `libplayos` C ABI, `playos-runtime` owns internal lifecycle transport and control IPC, and one isolated game process runs at a time. The system remains immutable, the data partition remains writable, and the player never interacts with Linux as a desktop operating system.
