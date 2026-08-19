# PlayOS Post-MVP Roadmap

> Features added only after the core console lifecycle (v0.1.0 MVP) is stable and shipped.  
> **Cross-references:** [roadmap.md](roadmap.md) §Post-MVP, [architecture.md](architecture.md) §22

Each item is listed with its motivation, dependencies, and rough priority tier.

---

## Tier 1 — Near-Term (v0.2.x)

These features are most commonly requested and have direct dependencies on shipped MVP components.

### Wi-Fi (`playos-net`)
**Sprint:** [Sprint 16](sprints/Sprint-16.md) (`playos-net`)
**Motivation:** Users need network access for future store downloads, cloud saves, and updates.  
**Stack:** `wpa_supplicant` + `dhcpcd` (D-Bus-free) via a trusted `playos-net` bridge — no NetworkManager, no iwd  
**Scope:** Connect to WPA2/WPA3 networks; Wi-Fi settings screen in shell  
**Depends on:** Networking enabled in kernel config (deferred in MVP)  
**Options:** See [networking options](sprints/network-options.md) for the D-Bus trade-off and the D-Bus-free `wpa_supplicant` alternative.

### OTA System Updates + `playos-tools` Staging Helper
**Motivation:** [Sprint 11](sprints/Sprint-11.md) ships the *offline* A/B update engine (signed `.playosb` → inactive slot → `boot.json` → rollback) but no delivery path. The ROG Ally has no Wi-Fi in MVP, so updates arrive by sneakernet: download on a workstation, stage to USB/SD, copy into `/data/updates/`. No network is required to *apply* an update — network only affects how the bundle is *delivered*.
**Two phases:**
1. **`playos-tools` host helper (works now, no on-device Wi-Fi):** a workstation CLI that downloads a signed `.playosb`, verifies its signature, and stages it to USB/SD for copy into `/data/updates/`. This closes the delivery gap immediately after Sprint 11 without touching the Ally's network stack.
2. **On-device download (after `playos-net`):** the shell's "Check for Update" downloads the bundle over Wi-Fi directly into `/data/updates/`, reusing Sprint 11's `ApplyUpdate` IPC and `boot.json` contract unchanged — the update engine itself needs no modification.
**Depends on:** Sprint 11 (A/B update engine + `/data/updates/*.playosb` contract — MVP); phase 2 additionally depends on Wi-Fi (`playos-net`).
**Sprint:** not yet allocated (post-MVP). Phase 1 is a standalone `playos-tools` task; phase 2 is a follow-up to [Sprint 16](sprints/Sprint-16.md) (`playos-net`).

### Touch + On-Screen Keyboard (OSK)
**Motivation:** The ROG Ally touchscreen is currently inert (no `wl_touch` forwarding), and every text-entry flow (Wi-Fi passphrase, search, save naming, profiles) needs a keyboard.  
**Stack:** Touch/pointer via the Wayland seat (`wl_pointer`/`wl_touch` + `wlr_scene` hit-testing); text input via upstream `zwp_text_input_v3` (`wlr_text_input_v3`); OSK UI rendered by `playos-overlay` as a raylib component.  
**Scope:** Touch reaches the focused surface (`GetTouchPosition`); a single system OSK is invokable by both the shell and games and delivers `commit_string` to the focused client.  
**Depends on:** MVP input API stable; Sprint 7 overlay architecture; Sprint 8 gamepad-input precedent.  
**Sprint:** [Sprint 17](sprints/Sprint-17.md)

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

### NVIDIA Backend (Nouveau + NVK + Zink)
**Motivation:** Extend the Sprint 13 portability proof (PCI enumeration + `playos-platform-api` backend abstraction) to a third vendor. NVIDIA's *open-source* stack is musl-compatible, unlike the proprietary userspace, which is glibc-only and off-limits for PlayOS (ADR-0003 musl).  
**Stack:** Nouveau (in-kernel DRM/KMS) + Mesa NVK (Vulkan) + Zink (GL/GLES on Vulkan); signed GSP firmware for Turing+ (redistributable via linux-firmware).  
**Scope:** Add a `nouveau` DRM/KMS + NVK + Zink Buildroot config and a third `libplayos` backend variant. Requires the wlroots **Vulkan** renderer (current renderer is GLES2/EGL, and Nouveau's Gallium GL is weak — NVK→Zink is the mature path).  
**Depends on:** Vulkan Support (RADV / ANV) landing first, so the wlroots Vulkan renderer and WSI path exist; Sprint 13 backend abstraction proven on Intel.  
**Note:** Deferred past Sprint 13 deliberately — Intel (`i915`) is the cheapest portability proof, and Nouveau power/reclocking maturity lags `amdgpu`, a real battery concern for a handheld.

### PPSSPP (PSP) Emulator Sample via libretro
**Motivation:** Ship a real PSP ISO game as a Sample Application, launched from `playos-shell` through a libretro PPSSPP core — a strong end-to-end proof of the sample/shell/SDK story using a real, redistributable game payload.  
**Stack:** libretro PPSSPP core (`ppsspp_libretro`) — GPLv2, `hw_render=true`, requires OpenGL ES ≥ 2.0, `needs_fullpath=true`, plus its asset pack (`ppge_atlas.zim`, `lang/`, `flash0/`; redistributable, not a BIOS).  
**Scope:** A libretro frontend shipped as a `bin/game` sample that hands off the GL context to the core, feeds input through the `libplayos` controller ABI, and maps save data to `/data/saves/<game-id>/`.  
**Dependency to resolve:** GL-context hand-off. Raylib's `PLATFORM_PLAYOS` backend (`rcore_playos.c`) owns the EGL/GLES context; the frontend must either expose that context (small additive change) or drive raw EGL/GLES + `libplayos` directly, bypassing raylib for this one sample. This is a libretro requirement, not a raylib limitation — emulator frontends sit on raw GL, not a game library.  
**Depends on:** Stable game ABI (raylib backend + `libplayos`), SDK build profile that can cross-compile the libretro frontend, and the context-exposure decision above.

### VRR and HDR
**Motivation:** ROG Ally display supports high refresh rates; future external displays may support VRR/HDR  
**Stack:** DRM VRR (`drm_connector.vrr_capable`); KMS HDR metadata  
**Depends on:** AMD DC VRR support stable in chosen kernel version

### Game Developer SDK (`playos-sdk`)
**Sprint:** [Sprint 15](sprints/Sprint-15.md) (Game Developer SDK)
**Motivation:** Third parties can't build a PlayOS game today without running Buildroot. The game ABI requires musl (not glibc) and links the musl builds of `libplayos` (and `libraylib`), which only exist inside the Buildroot tree. A self-contained SDK lets developers compile on a regular x86_64 Ubuntu host and ship a runnable game.  
**Stack:** Prebuilt `x86_64-buildroot-linux-musl` toolchain + `libplayos`/`libraylib` headers and static libs + a CMake toolchain file / `pkg-config` files. An Alpine Linux (musl-only) base image is a natural foundation: it already emits musl binaries natively, so the SDK only needs to add `libplayos`/`libraylib`.  
**Scope:** Downloadable tarball (or container image) that turns standard `gcc`/`cmake` on an x86_64 Ubuntu host into a single `bin/game` + `manifest.json` + `assets/` artifact. Binary must be musl-linked (static where possible) and target x86_64.  
**Testing story:** Because graphics/input/audio come from raylib (already cross-platform) and only the thin `libplayos` surface is PlayOS-specific, the SDK should expose three build profiles so developers can test without hardware: (1) `device` — musl + `PLATFORM_PLAYOS` raylib backend + real `libplayos` (evdev), the shipped artifact; (2) `desktop` — native `gcc` + raylib's default desktop backend (X11/Wayland on Linux, Win32/GLFW on Windows) + a host `libplayos` shim that maps keyboard/gamepad to the controller ABI and no-ops lifecycle, so the game runs in a normal desktop window; (3) `emulator` — run the device build inside the PlayOS QEMU/container image for high-fidelity testing. The `libplayos` `stub` backend (`PLAYOS_BACKEND=stub`) already exists as the seed for the desktop shim.  
**Depends on:** Versioned public Platform API (MVP); stable game ABI (Raylib backend + `libplayos` ABI)

### C# Shell Reimplementation (Investigation)
**Motivation:** Assess whether re-implementing `playos-shell` in C# would reduce memory-safety/manual-parsing risk and speed UI iteration.  
**Status:** Assessment only — **not** a planned feature direction. The default recommendation is to *not* pursue a C# rewrite; only a bounded host-only de-risking spike is documented.  
**Decisive factors:** .NET-on-musl/NativeAOT risk (ADR-0003), Buildroot toolchain effort, and loss of the single Raylib backend (`rcore_playos.c`, ADR-0006).  
**Sprint:** [Sprint 18](sprints/Sprint-18.md)

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

### Marketplace (`playos-marketplace`)
**Motivation:** A content-economy layer for publishing, discovering, installing, and updating PlayOS applications, games, themes, and developer content; multiple store sources (official, community, OEM, private, LAN) rather than a single hard-coded store.  
**Assessment:** See [Sprint 19 — Marketplace Assessment](sprints/Sprint-19.md). The repo is currently an empty stub; its docs reference "Part X — Package Format" and "Part XI — Cloud and Marketplace" in the spec, which do not yet exist. This is **spec-blocked**, not code-blocked.  
**Package format:** `.gpk` (canonical in `playos-marketplace/AGENTS.md`; reconcile with the historical `.play` name below before implementation).  
**V1 boundary:** Free-content catalog and install only — no payments, DRM, or entitlement enforcement.  
**Depends on:** Wi-Fi (`playos-net`), SDK (Sprint 15), manifest signing (Sprint 12), atomic-install pattern (Sprint 11), and the missing Part X/Part XI spec chapters.

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
    └── OTA System Updates (on-device download)

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
