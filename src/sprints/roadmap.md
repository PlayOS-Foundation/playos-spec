# PlayOS Roadmap

> This document defines the MVP exit criteria and sprint delivery plan.  
> Sprint documents contain the executable work packages: acceptance criteria, key tasks, and test strategy.

---

## MVP Definition

The first meaningful PlayOS MVP is complete when all of the following are true on physical ROG Ally hardware:

| # | Criterion |
|---:|---|
| 1 | ROG Ally boots directly from UEFI into PlayOS |
| 2 | Linux kernel + initramfs are available as a UEFI-bootable EFI artifact |
| 3 | `playos-init` runs as PID 1 |
| 4 | `playos-compositor` permanently owns DRM/KMS and the Wayland session |
| 5 | `playos-shell` remains alive as the persistent controller-first UI |
| 6 | Compositor uses wlroots with AMDGPU, DRM/KMS, GBM, EGL, and Mesa |
| 7 | Shell renders through Wayland using the Raylib PlayOS backend |
| 8 | Shell and sample game consume the public `playos-platform-api` C ABI |
| 9 | Trusted launch, lifecycle transport, and compositor-control remain internal to `playos-runtime` |
| 10 | Shell requests game launch; `playos-init` spawns and supervises it |
| 11 | Compositor waits for game's first valid frame before switching foreground |
| 12 | Game renders with hardware acceleration and receives controller input |
| 13 | Reserved System button returns to PlayOS UI and backgrounds/pauses the game |
| 14 | Resume returns to the same running game without restarting it |
| 15 | Game outputs audio through ALSA |
| 16 | Clean exit and crash both return safely to the existing shell |
| 17 | Games and saves persist on a separate ext4 partition |
| 18 | System image is immutable |
| 19 | Recovery mode usable without accelerated graphics |

---

## Sprint Plan

| Sprint | Title | Primary Outcome |
|---:|---|---|
| [0](Sprint-0.md) | Build and UEFI Foundation | Reproducible Buildroot factory boots a minimal PlayOS EFI image in QEMU/OVMF |
| [1](Sprint-1.md) | `playos-init` and Minimal Boot Supervision | Real `playos-init` as PID 1 with versioned private control IPC skeleton |
| [2](Sprint-2.md) | Compositor Skeleton and Wayland Session | Minimal wlroots compositor with a Wayland session and one trusted fullscreen client |
| [3](Sprint-3.md) | ROG Ally Kernel and Device Bring-Up | Reliable USB boot, essential ROG Ally devices working, first Platform API input contract |
| [4](Sprint-4.md) | AMDGPU and Native DRM/KMS | Compositor permanently owns the Ally display via AMDGPU and DRM/KMS |
| [5](Sprint-5.md) | Raylib-Powered PlayOS Shell | Hardware-accelerated Raylib shell consuming the public PlayOS Platform API |
| [6](Sprint-6.md) | Persistent Storage and Game Discovery | Persistent ext4 storage, Platform API paths, shell-visible game discovery |
| [7](Sprint-7.md) | Game Launch, Lifecycle, System Button, and Overlay | Complete console lifecycle: launch, overlay, background, resume, crash recovery |
| [8](Sprint-8.md) | ALSA Audio | Reliable ALSA audio with safe public controls across lifecycle transitions |
| [9](Sprint-9.md) | Power, Battery, Thermal, and Suspend Foundations | Safe power behavior exposed through a restricted public Platform API |
| [10](Sprint-10.md) | Installer and Internal-Disk Deployment | Tested installation path from removable media to ROG Ally internal SSD |
| [11](Sprint-11.md) | Immutable Images and A/B Updates | Signed, atomic A/B system updates with automatic rollback |
| [11.6](Sprint-11.6.md) | Developer SSH (Dropbear) + Minimal Wired Network Bring-Up | USB-C Ethernet SSH (key auth) for on-device debugging; full Wi-Fi stays Sprint 16 |
| [12](Sprint-12.md) | Security Hardening | Hardened boundary between public Platform API, trusted runtime control, and games |
| [13](Sprint-13.md) | Intel Expansion | Architecture and Platform API backend portable to Intel graphics |
| [14](Sprint-14.md) | Production Readiness | Signed preview release with versioned public Platform API |
| [15](Sprint-15.md) | Game Developer SDK | Self-contained `playos-sdk` (musl toolchain + `libplayos`/`libraylib`) with device/desktop/emulator testing |
| [16](Sprint-16.md) | `playos-net` (Wi-Fi) | D-Bus-free Wi-Fi (`wpa_supplicant` + `dhcpcd` + `playos-net` bridge) driven through `playos-runtime` |
| [17](Sprint-17.md) | Touch Input + On-Screen Keyboard (OSK) | Touch end-to-end (compositor → raylib backend) plus a reusable system OSK |
| [18](Sprint-18.md) | C# Shell Reimplementation Assessment (Post-MVP Spike) | Feasibility assessment only — no C# shell implemented |
| [19](Sprint-19.md) | Marketplace Assessment (Post-MVP) | Assessment and spec-first sequencing only — no marketplace code |
| [20](Sprint-20.md) | Native Media & Browser Client Strategy (Post-MVP) | Assessment of native Spotify/YouTube/YouTube Music/browser clients — Netflix out of scope |
| [21](Sprint-21.md) | Multiple Local User Profiles (Post-MVP) | Assessment/design of console-style local profiles with per-profile isolated saves/settings — no implementation |
| [22](Sprint-22.md) | LVGL Shell UI Spike (Post-MVP) | Gated LVGL-via-raylib texture spike with controller navigation + go/no-go — no shell port |

---

## Execution Rules

1. Sprints follow dependency order unless an ADR explicitly changes the sequence.
2. A sprint begins only after its required predecessor exit criteria are satisfied.
3. Each sprint must end with a **demonstrable and testable** system outcome.
4. Architecture changes discovered during implementation must be captured in `playos-spec`.
5. Cross-sprint API changes require a version bump and compatibility review.

---

## Post-MVP Roadmap

Add only after the core console lifecycle is stable:

- `playos-device` — hardware and power policy service
- Dropbear SSH — explicit Developer Mode only (minimal wired slice pulled forward in [Sprint 11.6](Sprint-11.6.md); full Wi-Fi remains [Sprint 16](Sprint-16.md))
- `playos-update` — PlayOS wrapper around the update engine
- OTA update delivery — `playos-tools` host helper (download + verify + stage to USB/SD) first, then on-device download after Sprint 16 (`playos-net`)
- `playos-input` service — remapping, virtual gamepads, gyro, haptics
- Dedicated audio service — mixing, notifications over games
- Bluetooth
- Fast, fully qualified suspend/resume
- Rear-button and special-button support
- Screenshots and screen recording
- Vulkan (RADV)
- VRR and HDR
- External-display profiles
- Download manager and store integration
- Cloud saves and user accounts
- Multiple local profiles
- Signed `.play` content packages
- Delta updates
- Telemetry (explicit user consent only)

---

*See [`architecture.md`](architecture.md) for the full system design.*
