# Summary

[Introduction](README.md)

---

# Architecture

- [System Architecture](architecture.md)
- [Platform API — libplayos](platform-api.md)
- [Runtime IPC Protocol](runtime-ipc.md)
- [Wayland Protocol](wayland-protocol.md)
- [Security Model](security-model.md)

---

# Component Specifications

- [playos-init (PID 1)](playos-init-spec.md)
- [playos-compositor](playos-compositor-spec.md)
- [playos-shell](playos-shell-spec.md)
- [playos-overlay](playos-overlay-spec.md)

---

# Build & Development

- [Build Guide](build-guide.md)
- [Kernel Configuration](kernel-config.md)
- [Developer Environment](dev-environment.md)
- [Testing Strategy](testing.md)
- [Installation Guide](installation-guide.md)
- [Partition Layout](partition-layout.md)

---

# Delivery

- [Roadmap & MVP Criteria](sprints/roadmap.md)
- [Sprint 0 — Build and UEFI Foundation](sprints/Sprint-0.md)
- [Sprint 1 — `playos-init` and Minimal Boot Supervision](sprints/Sprint-1.md)
- [Sprint 2 — Compositor Skeleton and Wayland Session](sprints/Sprint-2.md)
- [Sprint 2.5 — Cross-Sprint Audit Remediation](sprints/Sprint-2.5.md)
- [Sprint 3 — ROG Ally Kernel and Device Bring-Up](sprints/Sprint-3.md)
- [Sprint 4 — AMDGPU and Native DRM/KMS](sprints/Sprint-4.md)
- [Sprint 5 — Raylib-Powered PlayOS Shell](sprints/Sprint-5.md)
- [Sprint 5.5 — Shell → Raylib 6.0 Migration](sprints/Sprint-5.5.md)
- [Sprint 5.6 — ReposCleanUp](sprints/Sprint-5.6.md)
- [Sprint 6 — Persistent Storage and Game Discovery](sprints/Sprint-6.md)
- [Sprint 7 — Game Launch, Lifecycle, System Button, and Overlay](sprints/Sprint-7.md)
- [Sprint 8 — ALSA Audio](sprints/Sprint-8.md)
- [Sprint 9 — Power, Battery, Thermal, and Suspend Foundations](sprints/Sprint-9.md)
- [Sprint 9.5 — Display Brightness Control](sprints/Sprint-9.5.md)
- [Sprint 10 — Installer and Internal-Disk Deployment](sprints/Sprint-10.md)
- [Sprint 11 — Immutable Images and A/B Updates](sprints/Sprint-11.md)
- [Sprint 11.5 — Pivot-to-Squashfs Boot and A/B Validation](sprints/Sprint-11.5.md)
- [Sprint 11.6 — Developer SSH (Dropbear) + Minimal Wired Network Bring-Up](sprints/Sprint-11.6.md)
- [Sprint 12 — Security Hardening](sprints/Sprint-12.md)
- [Sprint 13 — Intel Expansion](sprints/Sprint-13.md)
- [Sprint 14 — Production Readiness](sprints/Sprint-14.md)
- [Sprint 15 — Game Developer SDK](sprints/Sprint-15.md)
- [Sprint 16 — `playos-net` (Wi-Fi)](sprints/Sprint-16.md)
- [Sprint 17 — Touch Input + On-Screen Keyboard (OSK)](sprints/Sprint-17.md)
- [Sprint 18 — C# Shell Reimplementation Assessment (Post-MVP Spike)](sprints/Sprint-18.md)
- [Sprint 19 — Marketplace Assessment (Post-MVP)](sprints/Sprint-19.md)
- [Sprint 20 — Native Media & Browser Client Strategy (Post-MVP)](sprints/Sprint-20.md)
- [Sprint 21 — Multiple Local User Profiles (Post-MVP)](sprints/Sprint-21.md)
- [Sprint 22 — LVGL Shell UI Spike (Post-MVP)](sprints/Sprint-22.md)
- [Post-MVP Features](post-mvp.md)
- [Networking Options — Wi-Fi & Bluetooth](sprints/network-options.md)
- [ROG Ally Input Handling](sprints/input-handling.md)

---

# Architecture Decision Records

- [ADR-0001 — Repository Structure](adr/ADR-0001-repository-structure.md)
- [ADR-0002 — IPC Transport](adr/ADR-0002-ipc-transport.md)
- [ADR-0003 — libc Choice (musl)](adr/ADR-0003-libc-choice.md)
- [ADR-0004 — Compositor Framework (wlroots)](adr/ADR-0004-compositor-framework.md)
- [ADR-0005 — Update Engine (RAUC)](adr/ADR-0005-update-engine.md)
- [ADR-0006 — UI Framework (Raylib)](adr/ADR-0006-ui-framework.md)
- [ADR-0007 — Audio Stack (ALSA)](adr/ADR-0007-audio-stack.md)
- [ADR-0008 — GPU Discovery](adr/ADR-0008-gpu-discovery.md)

---

# Reference

- [Original Design Notes](ideas.md)
