# PlayOS Post-MVP Roadmap

> Features added only after the core console lifecycle (v0.1.0 MVP) is stable and shipped.  
> **Cross-references:** [roadmap.md](roadmap.md) §Post-MVP, [architecture.md](architecture.md) §22

Each item is listed with its motivation, dependencies, and rough priority tier.

---

## Tier 1 — Near-Term (v0.2.x)

These features are most commonly requested and have direct dependencies on shipped MVP components.

### Wi-Fi (`playos-net`)
**Motivation:** Users need network access for future store downloads, cloud saves, and updates.  
**Stack:** `iwd` (iNet Wireless Daemon) — minimal, no NetworkManager  
**Scope:** Connect to WPA2/WPA3 networks; Wi-Fi settings screen in shell  
**Depends on:** Networking enabled in kernel config (deferred in MVP)

### SSH Developer Mode (Dropbear)
**Motivation:** Developers need remote access for debugging without a physical serial connection.  
**Policy:** SSH is present only in Developer Mode, which requires explicit user opt-in. Not enabled by default. Absent from retail builds.  
**Stack:** Dropbear — small SSH server  
**Depends on:** Wi-Fi

### Input Service (`playos-input`)
**Motivation:** Controller remapping, gyroscope support, haptic feedback, multi-controller assignment.  
**Scope:** Dedicated service abstracts raw evdev; exposes logical mappings; supports profiles  
**Replaces:** Direct evdev reading in `playos-platform-api`  
**Depends on:** MVP input API stable

### Full Suspend/Resume
**Motivation:** Battery life on portable device; lid-close expected behavior  
**Scope:** Full S3 or s2idle suspend; reliable resume without display artifacts  
**Blockers:** AMD AMDGPU suspend/resume on the ROG Ally requires validated kernel + firmware path. Do not ship until confirmed stable (no corrupted display on resume, no GPU hang).  
**Depends on:** AMD P-state + firmware validation

### Rear Buttons and Special Buttons
**Motivation:** ROG Ally has macro/custom buttons not mapped in MVP  
**Scope:** Map to logical PlayOS actions or user-configurable macros via `playos-input`  
**Depends on:** `playos-input` service

---

## Tier 2 — Medium-Term (v0.3.x)

### Dedicated Audio Service
**Motivation:** Multiple simultaneous audio owners (shell notifications over game music), Bluetooth audio, per-application volume  
**Stack:** PipeWire or a custom lightweight mixer  
**Scope:** Shell sound effects play over game audio; overlay audio feedback; notification sounds  
**Replaces:** Direct ALSA exclusivity in MVP  
**Depends on:** Stable lifecycle events

### Bluetooth
**Motivation:** Wireless controllers, headsets, keyboards  
**Stack:** BlueZ (minimal config — no A2DP profile unless combined with audio service)  
**Depends on:** `playos-net` (shares kernel network stack), audio service (for BT audio)

### Screenshots and Screen Recording
**Motivation:** Share game moments  
**Stack:** wlroots `wlr-screencopy` protocol for capture; encode with `ffmpeg` or similar  
**Policy:** Game must opt-in or be allowed by PlayOS policy (no silent capture)  
**Depends on:** Compositor update to expose `wlr-screencopy` to trusted clients

### Vulkan Support (RADV / ANV)
**Motivation:** Modern games use Vulkan; future game store requires Vulkan support  
**Stack:** Mesa RADV (AMD); Mesa ANV (Intel)  
**Scope:** Add RADV to the ROG Ally Buildroot config; add Vulkan WSI path to the Raylib backend or expose raw Vulkan surface to games  
**Depends on:** MVP graphics stack stable; no compositor ownership change needed

### VRR and HDR
**Motivation:** ROG Ally display supports high refresh rates; future external displays may support VRR/HDR  
**Stack:** DRM VRR (`drm_connector.vrr_capable`); KMS HDR metadata  
**Depends on:** AMD DC VRR support stable in chosen kernel version

### Game Developer SDK (`playos-sdk`)
**Motivation:** Third parties can't build a PlayOS game today without running Buildroot. The game ABI requires musl (not glibc) and links the musl builds of `libplayos` (and `libraylib`), which only exist inside the Buildroot tree. A self-contained SDK lets developers compile on a regular x86_64 Ubuntu host and ship a runnable game.  
**Stack:** Prebuilt `x86_64-buildroot-linux-musl` toolchain + `libplayos`/`libraylib` headers and static libs + a CMake toolchain file / `pkg-config` files. An Alpine Linux (musl-only) base image is a natural foundation: it already emits musl binaries natively, so the SDK only needs to add `libplayos`/`libraylib`.  
**Scope:** Downloadable tarball (or container image) that turns standard `gcc`/`cmake` on an x86_64 Ubuntu host into a single `bin/game` + `manifest.json` + `assets/` artifact. Binary must be musl-linked (static where possible) and target x86_64.  
**Depends on:** Versioned public Platform API (MVP); stable game ABI (Raylib backend + `libplayos` ABI)

---

## Tier 3 — Long-Term (v1.0+)

### Cloud Saves and User Accounts
**Motivation:** Save portability across devices; online game library  
**Scope:** Account authentication (OAuth2 or PlayOS account service); save sync on game launch/exit  
**Depends on:** Wi-Fi, network service, backend infrastructure

### Store Integration and Download Manager
**Motivation:** Users install games from a store without manual file transfer  
**Scope:** Browse, purchase, download, install, update games  
**API addition:** `playos_store.h` for store query and install progress  
**Depends on:** Wi-Fi, signed `.play` packages, cloud saves

### Signed `.play` Content Packages
**Motivation:** Integrity verification and atomic installation of games  
**Format:** Signed archive with: `manifest.json`, binary, assets, content hash tree  
**Replaces:** Plain directory installs  
**Depends on:** Manifest signing (Sprint 12 foundations)

### Delta Updates (Games and System)
**Motivation:** Reduce download size for incremental game and system updates  
**Stack:** `casync` or `bsdiff` or a PlayOS-specific delta tool  
**Depends on:** Store integration, A/B system updates (MVP)

### Multiple Local User Profiles
**Motivation:** Family devices; per-user settings, saves, and game libraries  
**Scope:** Profile selection at boot or via shell; per-profile `/data/saves/<profile>/<game-id>/`  
**Depends on:** Storage layout stable (MVP)

### External Display Profiles
**Motivation:** Docking station or TV output; ROG Ally supports USB-C DisplayPort  
**Scope:** Output detection via hotplug; resolution/refresh profile selection; orientation  
**Depends on:** Compositor output management (MVP compositor handles hotplug)

### Telemetry (Opt-In Only)
**Motivation:** Understand crash patterns and device health  
**Policy:** Explicit user opt-in required. No telemetry in builds where the user has not consented. Anonymous only.  
**Scope:** Crash reports, game launch/exit events, performance metrics  
**Depends on:** Wi-Fi, user accounts

### Performance Overlay (In-Game HUD)
**Motivation:** Show FPS, GPU/CPU usage, temperatures while gaming  
**Scope:** Compositor renders a lightweight HUD overlay above the game  
**Depends on:** Overlay architecture (MVP)

---

## Post-MVP Feature Dependencies

```
Wi-Fi
    └── SSH Developer Mode
    └── Dedicated Audio Service (BT)
    └── Bluetooth
    └── Cloud Saves
    └── Store + Download Manager

Store + Download Manager
    └── Signed .play Packages
    └── Delta Updates

Input Service
    └── Rear Buttons
    └── Gyroscope
    └── Haptics

Audio Service
    └── Bluetooth Audio
    └── Notifications over Games

Suspend/Resume
    └── (requires AMD firmware validation — not a feature dependency)
```

---

## Items Never Planned

These are explicitly out of scope for PlayOS, even post-MVP:

- General-purpose Linux desktop environment
- X11 / Xwayland
- Browser-based shell or WebAssembly runtime
- Multi-GPU or hybrid-graphics rendering
- Running multiple games simultaneously
- Cloud gaming (streaming from remote server)
- Custom GPU driver or OpenGL implementation
