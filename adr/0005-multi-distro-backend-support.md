# Add Arch Linux as an Alternative Distro Backend

- **ADR:** 0005
- **Title:** Add Arch Linux as an Alternative Distro Backend
- **Status:** Accepted
- **Date:** 2026-07-28
- **Deciders:** PlayOS Foundation
- **Supersedes:** None (ADR-0004 remains active)

## Context

[ADR-0004](0004-use-alpine-linux-reference-os-base.md) established Alpine Linux as the sole reference OS base. It explicitly requires that a future Arch or other distro backend have its own proposal and maintained package recipes, image construction, init/service definitions, validation, and release lifecycle.

The old Arch prototype (commits 07b352c through 4521698) proved the vertical slice — compositor, shell, and ROG Ally hardware path all worked. That code was retired from active repositories per ADR-0004, with Git history preserved as the canonical archive.

Since ADR-0004, two motivations for an Arch backend have grown:

- **Handheld-optimized kernels.** CachyOS provides two kernel variants tuned for handheld gaming PCs (ROG Ally, Steam Deck, Legion Go). The `linux-cachyos` kernel uses the EEVDF scheduler for general use; `linux-cachyos-deckify` uses the BORE scheduler with handheld-specific patches (input, audio, TDP control).
- **glibc application compatibility.** Some games, runtimes, and third-party components assume glibc at the host level. An Arch backend provides native glibc without requiring a compatibility container boundary for every payload.

Alpine remains the reference implementation. Arch is added as an **alternative** backend — not a replacement — producing identical disk image and ISO artifacts through the same build entry point.

## Decision

PlayOS will support **two distro backends**: Alpine (reference) and Arch (alternative). Each has its own maintained profile, package recipes, init/service definitions, and release lifecycle.

### Distro Selection

The build system selects the backend via an environment variable:

```text
PLAYOS_DISTRO=alpine|arch
```

When unset, the default is `alpine` — the reference implementation defined in ADR-0004. Changing the variable is the only user-facing switch; all downstream artifact paths, image names, and test matrices are derived from it.

### Kernel Variant Selection (Arch Only)

For the Arch backend, the handheld kernel variant is selected via:

```text
PLAYOS_KERNEL_VARIANT=cachyos|deckify
```

| Variant | Kernel Package | Scheduler | Use Case |
|---|---|---|---|
| `cachyos` (default) | `linux-cachyos` | EEVDF | General Arch backend |
| `deckify` | `linux-cachyos-deckify` | BORE | Handheld-optimized (ROG Ally, Steam Deck, Legion Go) |

### CachyOS Repository Pinning

CachyOS repositories are pinned for reproducibility. The build pins a single mirror and exact kernel package versions rather than following a rolling `latest`. This avoids silent kernel ABI or config changes between builds.

### Init System

| Backend | Init System |
|---|---|
| Alpine | OpenRC |
| Arch | systemd (native) |

Systemd is the native init system for Arch. Systemd units replace OpenRC init scripts for the Arch backend. Both backends implement the same PlayOS service contracts — the boot sequence (UEFI → kernel → init → seatd → compositor → shell → async services) is identical in intent, differing only in the init-specific wiring.

### CachyOS znver4 Packages

The Arch backend uses CachyOS packages compiled with `znver4` optimization, targeting the AMD Z1 Extreme and similar Zen 4 mobile APUs found in current handhelds. This is a build-time optimization only; it does not change the package set or runtime behavior.

### Package Lists

**General Arch backend** (~30 packages):

```
base systemd mesa wayland seatd pipewire wireplumber
networkmanager iwd bluez linux-cachyos
```

**Handheld Arch backend** (~33 packages):

```
base systemd mesa wayland seatd pipewire wireplumber
networkmanager iwd bluez linux-cachyos linux-cachyos-deckify
asusctl
```

The `cachyos-handheld` meta-package was **rejected**. It pulls Plasma, Steam, Gamescope, and MangoHud — a desktop/gaming stack that contradicts PlayOS console-appliance goals. The explicit package list keeps the image minimal and auditable.

### Shared Abstraction Layer

A `shared/` directory contains distro-agnostic code that both backends consume:

| Concern | Contents |
|---|---|
| Partition layout | GPT scheme, partition sizes, filesystem types |
| Bootloader install | systemd-boot configuration |
| Firstboot logic | Device identity, hostname, SSH keys, data partition setup |
| Device profiles | Hardware-specific configurations (ROG Ally, Steam Deck, etc.) |
| fstab generation | Read-only root, writable data, tmpfs overlays |

Distro-specific code is isolated in `alpine/` and `arch/` directories with no cross-dependencies.

The existing `build-iso-ubuntu.sh` entry point becomes a distro dispatcher:

```sh
case "${PLAYOS_DISTRO:-alpine}" in
  alpine) source alpine/build.sh ;;
  arch)   source arch/build.sh ;;
  *)      echo "Unknown PLAYOS_DISTRO: ${PLAYOS_DISTRO}" >&2; exit 1 ;;
esac
```

Both backends produce identical artifact types: raw disk images and bootable ISOs.

### Relationship to ADR-0004

ADR-0004 is **not superseded**. Alpine remains the reference implementation. All new Platform API features, compositor changes, and shell work must be validated on Alpine first. Arch may lag behind Alpine in feature support.

## Consequences

### Positive

- Handheld-optimized kernels (CachyOS EEVDF and BORE variants) with validated patches for ROG Ally, Steam Deck, and Legion Go.
- Native glibc compatibility for applications, runtimes, and third-party components that assume it.
- Larger package ecosystem via Arch and AUR for non-core dependencies.
- Community familiarity with Arch/systemd lowers contribution barrier for some developers.
- Identical artifact types from both backends, reducing downstream integration cost.
- Shared abstraction layer prevents code duplication for partition layout, bootloader, firstboot, and device profiles.

### Negative

- Maintenance burden doubles: two init systems, two package managers, two kernel update cadences, two validation matrices.
- Systemd adds complexity compared to OpenRC for the Arch backend.
- glibc binary size is approximately 2× musl, increasing image size.
- Rolling-release nature of Arch requires more frequent integration testing even with pinned CachyOS kernels.
- Risk of divergence between backends if feature work is validated on only one.

### Risks and Mitigations

- **Alpine build must not break during refactor.** Every change that touches `shared/` must be validated with a full Alpine build and QEMU boot test. CI gates enforce this.
- **Divergent behavior between backends.** Shared abstraction layer and identical artifact format reduce surface area. Behavioral differences must be documented as backend-specific notes.
- **CachyOS kernel availability.** Pinning exact versions mitigates upstream churn. If CachyOS ceases maintenance, the Arch backend can fall back to `linux-zen` or `linux` with handheld patches applied separately.
- **Arch rolling-release instability.** Pinned CachyOS repositories and version-locked kernel packages provide a stable input set. Non-kernel packages may still drift; critical packages (mesa, wayland, seatd) should be version-pinned.
- **Systemd unit drift from OpenRC services.** Both must implement the same PlayOS service contracts. Integration tests verify boot sequence parity.

## Alternatives Considered

- **Arch-only (drop Alpine):** Rejected. Alpine's smaller userland, musl, and read-only/diskless workflows remain valuable for the reference implementation. ADR-0004's rationale still holds.
- **CachyOS `cachyos-handheld` meta-package:** Rejected. Pulls an opinionated desktop/gaming stack (Plasma, Steam, Gamescope, MangoHud) incompatible with PlayOS console-appliance design.
- **Arch with OpenRC instead of systemd:** Rejected. Systemd is Arch's native init system; replacing it with OpenRC recreates integration burden and loses community-tested service definitions.
- **Fedora/Universal Blue as second backend:** Deferred. Strong handheld enablement (Bazzite), but adds a third package manager (rpm-ostree) and image model without proven demand beyond what Arch provides.
- **Single build script per distro (no shared abstraction):** Rejected. Partition layout, bootloader, firstboot, and device profiles are distro-agnostic concerns. Duplicating them guarantees drift.

## Non-goals

This decision does not:

- make Arch the reference implementation (Alpine retains that role per ADR-0004);
- require all Platform API features to ship simultaneously on both backends;
- support more than two active distro backends without a new ADR;
- endorse CachyOS as an upstream dependency — only its kernels and znver4 packages are consumed;
- change the PlayOS compositor, shell, or runtime architecture;
- imply that Arch packages or systemd units are part of the portable Platform API.
