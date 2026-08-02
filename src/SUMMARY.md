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

---

# Delivery

- [Roadmap & MVP Criteria](sprints/roadmap.md)
- [Sprint 0 — Repository & Toolchain Bootstrap](sprints/Sprint-0.md)
- [Sprint 1 — Kernel & Minimal Root](sprints/Sprint-1.md)
- [Sprint 2 — DRM/KMS Compositor](sprints/Sprint-2.md)
- [Sprint 3 — Input & Button Interception](sprints/Sprint-3.md)
- [Sprint 4 — Shell Skeleton](sprints/Sprint-4.md)
- [Sprint 5 — libplayos API](sprints/Sprint-5.md)
- [Sprint 6 — Game Launch & Lifecycle](sprints/Sprint-6.md)
- [Sprint 7 — First-Frame & Focus Protocol](sprints/Sprint-7.md)
- [Sprint 8 — Overlay & System UI](sprints/Sprint-8.md)
- [Sprint 9 — Storage & Save Games](sprints/Sprint-9.md)
- [Sprint 10 — Audio](sprints/Sprint-10.md)
- [Sprint 11 — Power & Suspend](sprints/Sprint-11.md)
- [Sprint 12 — A/B Update Engine](sprints/Sprint-12.md)
- [Sprint 13 — Hardening & Security](sprints/Sprint-13.md)
- [Sprint 14 — MVP Validation](sprints/Sprint-14.md)
- [Post-MVP Features](post-mvp.md)

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
