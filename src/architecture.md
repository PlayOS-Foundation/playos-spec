# PlayOS Architecture Reference

> **Version:** 2.4  
> **Device:** ROG Ally (AMD/AMDGPU primary)  
> **Source of truth:** [`ideas.md`](ideas.md) — this document is the distilled reference.

---

## Table of Contents

1. [Defining Principle](#1-defining-principle)
2. [System Diagram](#2-system-diagram)
3. [Repository Map](#3-repository-map)
4. [Process Model](#4-process-model)
5. [Boot Sequence](#5-boot-sequence)
6. [Component Responsibilities](#6-component-responsibilities)
7. [Console Lifecycle State Machine](#7-console-lifecycle-state-machine)
8. [Game Launch Flow](#8-game-launch-flow)
9. [Input Routing](#9-input-routing)
10. [Graphics Stack](#10-graphics-stack)
11. [Audio Stack](#11-audio-stack)
12. [Storage Layout](#12-storage-layout)
13. [Security Boundaries](#13-security-boundaries)
14. [Architectural Constraints (Non-Goals)](#14-architectural-constraints-non-goals)

---

## 1. Defining Principle

> PlayOS is a console operating environment built on a minimal Linux hardware layer.
> `playos-init` owns processes, `playos-compositor` owns display and focus,
> `playos-shell` owns the user experience, `playos-platform-api` owns the public
> `libplayos` C ABI, `playos-runtime` owns internal lifecycle transport and control IPC,
> and one isolated game process runs at a time. The system remains immutable, the data
> partition remains writable, and the player never interacts with Linux as a desktop
> operating system.

**Key axioms:**

- Linux is the hardware layer, not the product.
- The system image is immutable; only the data partition is writable.
- The compositor permanently owns DRM/KMS — no handoffs to games.
- One game runs at a time; `playos-shell` always stays alive.
- Games target `playos-platform-api`; they never touch compositor or kernel internals.

---

## 2. System Diagram

```
UEFI Firmware
    │
    ▼
Linux EFI-stub kernel  ◄── embedded initramfs
    │
    ▼
Linux Kernel
    ├── EFI / ACPI / PCIe / IOMMU
    ├── AMDGPU / DRM/KMS
    ├── USB / HID / evdev
    ├── ALSA
    ├── NVMe / ext4 / FAT
    └── battery / thermal / power
    │
    ▼
PlayOS Runtime (all in initramfs)
    ├── playos-init              PID 1, process supervisor
    ├── playos-compositor        wlroots compositor, DRM/KMS owner
    │       ├── playos-shell     persistent Raylib UI (Wayland client)
    │       ├── playos-overlay   trusted system overlay (Wayland client)
    │       └── active-game      one isolated Wayland game process
    ├── libplayos                public C ABI (from playos-platform-api)
    ├── playos-runtime           internal IPC and lifecycle transport
    └── Mesa / Wayland / ALSA   platform libraries
    │
    ▼
Persistent Data Partition (/data)
    games / saves / cache / log / updates / config
```

---

## 3. Repository Map

| Repository | Owns |
|---|---|
| `playos-spec` | Architecture, public contracts, ADRs, schemas, roadmap |
| `playos-init` | PID 1 process supervisor, boot lifecycle, storage mount, game launch/supervision |
| `playos-platform-api` | Public `libplayos` C ABI, C++ wrappers, engine adapters |
| `playos-runtime` | Internal IPC, lifecycle transport, private Wayland protocols, OS integration |
| `playos-compositor` | wlroots compositor, DRM/KMS, surface/focus/input policy |
| `playos-shell` | Raylib controller-first shell and trusted UX |
| `playos-refdistro` | Buildroot integration, kernel config, image assembly |

**Dependency direction:**

```
playos-spec
    └── defines contracts for all implementation repos

playos-runtime  ◄────────────────────────────  playos-compositor
    ▲                                                 ▲
    │                                                 │ (private control IPC)
playos-platform-api                          playos-shell (trusted client)
    ▲
    │
games / playos-shell (public API consumers)

playos-refdistro  ──  pins and assembles all runtime components
```

**Rules:**
- Public application ABI lives only in `playos-platform-api`.
- Private IPC/protocol definitions live only in `playos-runtime`.
- DRM/KMS and compositor implementation lives only in `playos-compositor`.
- `playos-refdistro` packages and pins; it does not redefine contracts.

---

## 4. Process Model

```
playos-init  (PID 1)
    │
    ├── playos-compositor        (DRM/KMS owner, Wayland display)
    │       ├── playos-shell      (trusted Wayland client, always alive)
    │       └── playos-overlay    (trusted Wayland client, shown on demand)
    │
    └── active-game               (isolated Wayland client, one at a time)
```

- `playos-init` spawns and supervises `playos-compositor`, `playos-shell`, `playos-overlay`, and `active-game`.
- The compositor owns the Wayland display and surface presentation for its trusted clients; it does not spawn processes.

---

## 5. Boot Sequence

| Step | Actor | Action |
|---:|---|---|
| 1 | UEFI | Loads `/EFI/BOOT/BOOTX64.EFI` |
| 2 | EFI stub | Transfers control to the Linux kernel |
| 3 | Kernel | Initializes hardware: ACPI, PCIe, GPU, USB, ALSA, NVMe |
| 4 | Kernel | Unpacks embedded initramfs into RAM |
| 5 | Kernel | Starts `/init` → `playos-init` as PID 1 |
| 6 | `playos-init` | Mounts `/dev`, `/proc`, `/sys`, `/run` |
| 7 | `playos-init` | Discovers and mounts the PlayOS data partition |
| 8 | `playos-init` | Starts `playos-compositor` |
| 9 | `playos-compositor` | Initializes wlroots backend, DRM/KMS, renderer, Wayland socket |
| 10 | `playos-init` | Launches `playos-shell` and `playos-overlay` with trusted identity |
| 11 | `playos-shell` | Maps fullscreen surface; shows game library |
| → | User | Selects a game |
| 12 | `playos-shell` | Sends `LaunchGame(game_id)` over control IPC |
| 13 | `playos-init` | Validates manifest; spawns game process |
| 14 | `playos-compositor` | Waits for game's first valid committed frame |
| 15 | `playos-compositor` | Switches foreground from shell → game |

**First-frame rule:** The compositor never switches to the game surface until it receives a real committed buffer. This prevents a black-screen transition during game initialization.

---

## 6. Component Responsibilities

### `playos-init`
**Owns:** Boot lifecycle, process supervision, storage mount, game launch/kill, reboot/shutdown/recovery.  
**Does NOT own:** Surfaces, focus, rendering, network, game-specific logic.

### `playos-compositor`
**Owns:** DRM/KMS, Wayland socket and display, surface z-order and focus, trusted client identity, reserved system input, overlay stacking, lifecycle state transitions, crash recovery.  
**Does NOT own:** Game installation, save management, process spawning.

### `playos-shell`
**Owns:** Persistent console UI, controller-first navigation, game discovery, launch requests via control IPC, settings, Raylib rendering.  
**Behavior while game is running:** Remains alive; stops or throttles rendering; available to show system UI.

### `playos-overlay`
**Owns:** Quick menu, volume/brightness HUD, power menu, notifications, virtual keyboard.  
May later merge into a multi-surface `playos-shell` backend.

### `playos-platform-api` / `libplayos`
**Owns:** Public, engine-agnostic C ABI. Stable across versions. Exposes lifecycle events, storage paths, device info, logical input, audio/display/power queries, structured logging.  
**Does NOT expose:** Compositor internals, privileged IPC, or DRM handles.

### `playos-runtime`
**Owns:** Internal IPC protocol definitions, lifecycle event transport, private Wayland protocol XML, restricted client libraries for trusted components, OS integration helpers.  
**Does NOT own:** DRM/KMS policy or the compositor implementation.

### Active game process
**Owns:** Its address space, Wayland surface, audio streams, input stream, save/cache directories.  
**Prohibited from:** Becoming DRM master, reconfiguring displays, mounting filesystems, modifying the system image, synthesizing reserved input, connecting to privileged IPC endpoints.

---

## 7. Console Lifecycle State Machine

```
SHELL_FOREGROUND
    │
    │  launch accepted by playos-init
    ▼
GAME_STARTING
    │
    │  first valid game frame committed
    ▼
GAME_FOREGROUND ◄─────────────────────────────────┐
    │                                              │
    │  PLAYOS_BUTTON_SYSTEM pressed                │ Resume
    ▼                                              │
PLAYOS_UI_FOREGROUND_WITH_GAME_BACKGROUND ─────────┘
    │
    │  Quit selected
    ▼
TERMINATING_GAME
    │
    ▼
SHELL_FOREGROUND ◄── game exits cleanly or crashes (any state)
```

**This state machine is a core PlayOS contract and must be explicitly tested.**

State transition triggers:

| From | Event | To |
|---|---|---|
| `SHELL_FOREGROUND` | `playos-init` accepts launch | `GAME_STARTING` |
| `GAME_STARTING` | First valid game frame | `GAME_FOREGROUND` |
| `GAME_FOREGROUND` | `PLAYOS_BUTTON_SYSTEM` | `PLAYOS_UI_FOREGROUND_WITH_GAME_BACKGROUND` |
| `PLAYOS_UI_...` | Resume | `GAME_FOREGROUND` |
| `PLAYOS_UI_...` | Quit | `TERMINATING_GAME` → `SHELL_FOREGROUND` |
| `GAME_FOREGROUND` | Exit or crash | `SHELL_FOREGROUND` |

---

## 8. Game Launch Flow

```
playos-shell      →  LaunchGame(game_id)  →  playos-init control IPC
playos-init       →  validates manifest, permissions, one-game rule
playos-init       →  prepares save/cache paths, process group, lifecycle channel, launch identity
playos-init       →  spawns game with WAYLAND_DISPLAY + PlayOS env vars
playos-compositor →  matches client to expected launch identity
playos-compositor →  waits for first committed buffer
playos-compositor →  switches foreground from shell → game
playos-shell      →  remains alive, rendering throttled
```

**Launch responsibility split:**

| Component | Role in launch |
|---|---|
| `playos-shell` | Chooses and requests via restricted control IPC |
| `playos-init` | Validates, spawns, supervises, terminates |
| `playos-compositor` | Identifies surface; controls presentation |
| `playos-runtime` | Transports lifecycle and control messages |
| `playos-platform-api` | Exposes lifecycle events and safe services to the game |

---

## 9. Input Routing

```
Controller / keyboard / touch
    │
    ▼
Linux HID / evdev / libinput
    │
    ▼
playos-compositor
    ├── PLAYOS_BUTTON_SYSTEM  →  PlayOS only (never delivered to games)
    ├── overlay visible        →  playos-overlay
    ├── game foreground        →  active game
    └── otherwise              →  playos-shell
```

**Logical input constants (defined in `playos-platform-api`):**

```c
PLAYOS_BUTTON_SOUTH / EAST / WEST / NORTH
PLAYOS_BUTTON_START / SELECT
PLAYOS_BUTTON_SYSTEM          // reserved — not delivered to games
PLAYOS_BUTTON_QUICK_MENU      // reserved
PLAYOS_AXIS_LEFT_X / LEFT_Y
PLAYOS_AXIS_RIGHT_X / RIGHT_Y
PLAYOS_AXIS_LEFT_TRIGGER / RIGHT_TRIGGER
```

---

## 10. Graphics Stack

```
Raylib shell or game
    │  Wayland / PlayOS Raylib backend (rcore_playos.c)
    ▼
Wayland protocol
    │
    ▼
playos-compositor + wlroots
    │  GBM / EGL / OpenGL ES renderer
    ▼
DRM/KMS + AMDGPU
    │
    ▼
Display
```

**AMD (primary):** `amdgpu` kernel driver → Mesa RadeonSI → libdrm / GBM / EGL  
**Intel (later):** `i915` / `xe` → Mesa Iris → libdrm / GBM / EGL  
**Vulkan:** Deferred (RADV / ANV added after AMD baseline is stable)

**Direct scanout:** When a game's fullscreen buffer is compatible with the DRM output plane, the compositor attempts to assign it directly, skipping composition. Falls back automatically when an overlay is visible or format/scaling prevents it.

**Recovery graphics:** SimpleDRM or firmware framebuffer fallback for recovery mode without accelerated graphics.

---

## 11. Audio Stack

```
Raylib audio
    │  PlayOS audio backend
    ▼
ALSA PCM
    │
    ▼
Kernel ALSA driver (HDA / SoC / ACP)
    │
    ▼
Built-in speakers / headphones
```

MVP policy:
- Stereo PCM only; no PulseAudio or PipeWire.
- One foreground audio owner: game while foreground, shell otherwise.
- Shell mutes/stops audio when game becomes foreground; resumes when game exits.

---

## 12. Storage Layout

**Partition model (production A/B):**

```
GPT disk
├── Partition 1: EFI System Partition   FAT32     BOOTX64.EFI
├── Partition 2: PlayOS system A        immutable
├── Partition 3: PlayOS system B        immutable
└── Partition 4: PlayOS data            ext4      writable
```

**Data partition (`/data`):**

```
/data/
    games/<game-id>/          manifest.json, bin/, assets/, shaders/, licenses/
    saves/<game-id>/          profiles/, autosaves/, settings/
    cache/<game-id>/          shaders/, compiled-assets/, temporary/
    resources/
    downloads/
    log/
    updates/
    screenshots/
    config/
    profiles/
```

**Rules:**
- PlayOS must never silently format an unknown disk.
- Factory reset operates on `/data` only; the immutable system slots are untouched.
- First-boot provisioning requires explicit confirmation before creating filesystems.

---

## 13. Security Boundaries

```
┌────────────────────────────────────────────────────────┐
│  Trusted system components                             │
│  playos-init (root)                                    │
│  playos-compositor (display/input caps)                │
│  playos-shell / playos-overlay (service user)          │
│                                                        │
│  ← communicate via restricted playos-runtime IPC →    │
└────────────────────────────────────────────────────────┘
                          │
              public libplayos C ABI
                          │
┌────────────────────────────────────────────────────────┐
│  Untrusted                                             │
│  active-game (unprivileged game identity)              │
│  per-title save and cache directories only             │
│  no DRM primary nodes                                  │
│  no mount, format, or kernel-module access             │
│  no reserved input synthesis                           │
│  no direct compositor or privileged IPC access         │
└────────────────────────────────────────────────────────┘
```

**Hardening roadmap:** capabilities → seccomp → Landlock → namespaces → signed manifests → Secure Boot (signed EFI + kernel + initramfs + A/B metadata).

---

## 14. Architectural Constraints (Non-Goals)

The following are explicitly excluded from PlayOS v1:

| Excluded | Reason |
|---|---|
| Desktop environment | Console OS; Linux is the hardware layer only |
| X11 / Xwayland | Wayland-only |
| systemd | Custom `playos-init` owns lifecycle |
| Display / login manager | Direct boot to shell |
| Containers | Not needed for single-game model |
| Conventional package manager | Immutable system image |
| Custom Linux kernel | Upstream LTS with ROG Ally config |
| Custom GPU driver / OpenGL | Mesa / AMDGPU |
| Multiple simultaneous games | One-game process model |
| Multi-GPU / hybrid graphics | ROG Ally is single AMD GPU |
| Wi-Fi, Bluetooth, SSH, cloud saves | Post-MVP |
| Full suspend/resume | Post-MVP |
| HDR, VRR, recording, streaming | Post-MVP |
| libc other than musl | musl only |

---

*For sprint-by-sprint implementation detail, see [`roadmap.md`](roadmap.md) and the individual `Sprint-N.md` files.*  
*For the public API contract, see [`platform-api.md`](platform-api.md).*  
*For internal IPC definitions, see [`runtime-ipc.md`](runtime-ipc.md).*
