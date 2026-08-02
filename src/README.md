# PlayOS

> A console operating environment for the ROG Ally — built on Linux, presented as a dedicated gaming console.

PlayOS boots directly from UEFI into a controller-first shell. A custom compositor permanently owns the display. One hardware-accelerated game runs at a time. The player never sees a Linux desktop, terminal, or login screen.

---

## What PlayOS Is

- A **minimal, immutable Linux system** that acts as a hardware enablement layer
- A **Wayland compositor** that owns DRM/KMS and enforces console display policy
- A **persistent Raylib shell** that is always alive, even while a game runs
- A **stable public C ABI** (`libplayos`) that games target instead of Linux internals
- A **six-repository project** with clear ownership boundaries

## What PlayOS Is Not

- A Linux distribution or desktop environment
- A general-purpose PC OS
- An emulation layer or compatibility shim
- A cloud gaming platform

---

## Documentation

### Architecture and Contracts

| Document | Description |
|---|---|
| [architecture.md](architecture.md) | System design, component diagrams, state machine, boot sequence |
| [platform-api.md](platform-api.md) | Public `libplayos` C ABI specification and versioning policy |
| [runtime-ipc.md](runtime-ipc.md) | Internal IPC protocol (launch, lifecycle, control) |
| [wayland-protocol.md](wayland-protocol.md) | Private PlayOS Wayland extensions |
| [security-model.md](security-model.md) | Trust boundaries, game restrictions, Secure Boot chain |

### Component Specifications

| Document | Description |
|---|---|
| [playos-init-spec.md](playos-init-spec.md) | PID 1 — boot, process supervision, storage, IPC |
| [playos-compositor-spec.md](playos-compositor-spec.md) | wlroots compositor — DRM/KMS, focus, state machine |
| [playos-shell-spec.md](playos-shell-spec.md) | Raylib shell — controller UI, game library, lifecycle |
| [playos-overlay-spec.md](playos-overlay-spec.md) | Trusted overlay — quick menu, notifications, power |

### Build and Development

| Document | Description |
|---|---|
| [build-guide.md](build-guide.md) | Buildroot setup, `br2-external` layout, `make` commands |
| [kernel-config.md](kernel-config.md) | Kernel subsystem requirements, ROG Ally configuration |
| [dev-environment.md](dev-environment.md) | QEMU/OVMF setup, developer iteration workflow |
| [testing.md](testing.md) | CI layers, physical device smoke tests |

### Delivery

| Document | Description |
|---|---|
| [roadmap.md](roadmap.md) | MVP criteria and sprint plan (Sprints 0–14) |
| [post-mvp.md](post-mvp.md) | Post-MVP feature roadmap |
| [Sprint-N.md](Sprint-0.md) | Sprint 0–14 work packages |

### Architecture Decision Records

| ADR | Decision |
|---|---|
| [ADR-0001](adr/ADR-0001-repository-structure.md) | Six-repository structure |
| [ADR-0002](adr/ADR-0002-ipc-transport.md) | Unix socket IPC transport |
| [ADR-0003](adr/ADR-0003-libc-choice.md) | musl libc only |
| [ADR-0004](adr/ADR-0004-compositor-framework.md) | wlroots as compositor foundation |
| [ADR-0005](adr/ADR-0005-update-engine.md) | RAUC for A/B updates |
| [ADR-0006](adr/ADR-0006-ui-framework.md) | Raylib for shell and game UI |
| [ADR-0007](adr/ADR-0007-audio-stack.md) | Direct ALSA for MVP audio |
| [ADR-0008](adr/ADR-0008-gpu-discovery.md) | PCI enumeration for GPU selection |

---

## Repository Map

| Repository | Role |
|---|---|
| `playos-spec` | Architecture, contracts, ADRs, roadmap, game developer docs |
| `playos-platform-api` | Public `libplayos` C ABI + Raylib backend |
| `playos-runtime` | Internal IPC, lifecycle transport, private Wayland protocols |
| `playos-compositor` | wlroots compositor, DRM/KMS, focus, input routing |
| `playos-shell` | Controller-first Raylib shell and game library |
| `playos-refdistro` | Buildroot integration, `playos-init`, image assembly, installer |

---

## Quick Start

```bash
# Build and run in QEMU
make setup
make qemu-config
make qemu-build
make qemu-run

# Build for ROG Ally (USB image)
make ally-config
make ally-build
make ally-usb-image
```

See [build-guide.md](build-guide.md) and [dev-environment.md](dev-environment.md) for full setup instructions.

---

## Primary Device

**ASUS ROG Ally** — AMD Ryzen Z1 / RDNA 3 APU  
First supported graphics stack: AMDGPU + Mesa RadeonSI  
Intel expansion: Sprint 13
