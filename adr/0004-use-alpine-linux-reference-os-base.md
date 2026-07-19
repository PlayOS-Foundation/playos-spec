# Use Alpine Linux as the PlayOS Reference OS Base

- **ADR:** 0004
- **Title:** Use Alpine Linux as the PlayOS Reference OS Base
- **Status:** Accepted
- **Date:** 2026-07-19
- **Deciders:** PlayOS Foundation
- **Supersedes:** [ADR-0003](0003-arch-linux-reference-runtime-base.md)

## Context

PlayOS needs a reference operating system for full Runtime Devices such as the ASUS ROG Ally. It must be a small, reproducible console appliance rather than a repackaged desktop or gaming distribution.

The reference OS must:

- reach the PlayOS Shell as soon as the GPU, input, seat, and compositor path is ready;
- avoid a display manager, desktop environment, and general-purpose login session;
- start networking, audio, Bluetooth, updates, indexing, cloud, and marketplace services outside the visual boot path;
- support x86_64 handheld PCs and future ARM64 devices;
- provide current Linux, firmware, Mesa, Wayland, wlroots, libinput, seatd, PipeWire, and BlueZ components;
- support read-only boot media and explicit persistent data;
- produce reproducible ISO and installed-disk images;
- keep distribution details outside the public PlayOS Platform API;
- allow PlayOS to own system policy without maintaining Linux from first principles.

The first vertical slice used Arch Linux, systemd, and archiso. It proved the compositor, shell, and ROG Ally hardware path, but coupled the image and service model to a rolling general-purpose distribution.

Alpine Linux provides a smaller appliance-oriented base using musl, BusyBox, apk, OpenRC, official image-building tools, and diskless/read-only modes. These properties align with PlayOS fast-boot and owned-policy goals.

## Decision

The PlayOS reference OS will use **Alpine Linux** as its upstream base.

| Area | Decision |
|---|---|
| Userland | Pinned Alpine stable release branch |
| C library | musl |
| Package manager | apk |
| Init and services | OpenRC |
| Image construction | Alpine `aports` and `mkimage` profiles |
| Device discovery | Alpine-supported udev-compatible management |
| Seat access | seatd |
| Graphics | Mesa, DRM/KMS, Wayland, and wlroots |
| Audio | PipeWire and WirePlumber |
| Network | NetworkManager with iwd where supported |
| Bluetooth | BlueZ |
| Boot media | Alpine read-only image and modloop/initramfs |
| Persistent state | Dedicated writable PlayOS data partition |
| Build environment | Pinned Alpine build root using Alpine-native tools |
| Reference hardware | ASUS ROG Ally |

Released images must not build directly from Alpine `edge`. Kernel, Mesa, firmware, and wlroots updates must be promoted deliberately after virtual and hardware validation.

OpenRC is the reference init system. PlayOS lifecycle concepts remain implementation-level contracts rather than systemd target names. The visual runlevel contains only services required for the first interactive shell frame. Background services enter a separate PlayOS asynchronous runlevel after compositor readiness.

```text
UEFI
  → bootloader
  → Linux kernel and Alpine initramfs
  → OpenRC sysinit/boot
  → GPU and input readiness
  → seatd
  → playos-compositor
  → playos-shell first frame
  → PlayOS asynchronous services
```

The PlayOS compositor remains the display owner. The shell remains a replaceable Wayland client. Alpine does not change ADR-0002 or merge the OS, runtime, compositor, and shell.

## Compatibility Boundary

Native PlayOS reference components must build and run against musl.

Applications must not assume the host exports glibc. Portable PlayOS packages should use supported PlayOS SDK/runtime contracts and declare ABI requirements.

Third-party games requiring glibc, Steam Runtime, or another userspace must run through an explicit compatibility runtime or container boundary. The reference OS may provide `gcompat` for narrow cases, but host-wide glibc emulation is not part of the base ABI.

## Repository Ownership

- `playos-spec` defines the portable contract and records this decision.
- `playos-refdistro` owns Alpine profiles, APK packaging, OpenRC services, boot configuration, and image tests.
- `playos-runtime` owns application supervision, runtime services, and the compositor, but remains distribution-independent.
- `playos-reference-devices` owns Alpine hardware bring-up material and device profiles.
- `playos-shell` remains a Wayland client and does not own boot policy.

## Implementation and Archive Policy

Alpine is the only active reference-distro profile. The former Arch profile, package lists, build scripts, and systemd units are removed from active repositories. Git history is the canonical archive and must not be rewritten to hide the Arch phase.

Distribution-neutral findings—especially ROG Ally firmware, graphics, input, networking, and boot observations—should be extracted into Alpine documentation and validation requirements. Arch-specific fixes must be revalidated and implemented with Alpine-native packages and tools rather than copied.

A future Arch or other distro backend requires a new proposal and its own maintained package recipes, image construction, init/service definitions, validation, and release lifecycle. Recovering files from history alone does not make that backend supported.

## Consequences

### Positive

- Smaller and more auditable reference userland.
- Alpine diskless and read-only workflows align with a console appliance.
- apk and pinned Alpine branches support controlled image inputs.
- OpenRC provides a compact service model.
- musl exposes accidental host ABI coupling.
- x86_64 and ARM64 share the same upstream model.
- PlayOS owns console policy while reusing an established distribution.

### Negative

- Proven capabilities from the Arch prototype must be reimplemented and revalidated with Alpine-native tooling.
- Native components may require musl portability fixes.
- Some proprietary games assume glibc.
- Package versions may differ from the first prototype.
- Handheld kernel, firmware, suspend, and input support require renewed validation.
- OpenRC readiness and resource controls must replace systemd units, targets, and slices.

### Risks and Mitigations

- **glibc-only payloads:** isolate them in declared compatibility runtimes.
- **hardware support gaps:** pin tested releases and promote kernel, Mesa, firmware, or backports through CI and hardware gates.
- **loss of prototype knowledge:** retain Git history and extract distribution-neutral findings into current documentation and tests.
- **slow services delaying UI:** keep them out of the visual runlevel.
- **distribution assumptions entering runtime code:** test components on musl and keep packaging/init logic in `playos-refdistro`.
- **edge instability:** prohibit unpinned edge inputs in releases.

## Alternatives Considered

- **Arch Linux:** proved the vertical slice and offers current packages, but its rolling and archiso/systemd model is no longer selected.
- **Fedora/Universal Blue and Bazzite:** strong enablement but inherit larger desktop/gaming policy.
- **Ubuntu/Debian:** broad compatibility but a larger base and slower graphics cadence.
- **Tiny Core Linux:** too unconventional and incomplete for the required stack.
- **postmarketOS:** useful Alpine device-enablement patterns, but PlayOS is not adopting its mobile product policy.
- **Buildroot or Yocto:** maximum control with substantially more maintenance.
- **Alpine with systemd:** rejected because it replaces Alpine's native service model and recreates integration burden.

## Non-goals

This decision does not:

- make Alpine part of the portable Platform API;
- require other PlayOS implementations or SDK Targets to use Alpine;
- turn PlayOS into a general-purpose Alpine desktop;
- guarantee arbitrary glibc binaries run directly on the host;
- replace the PlayOS compositor with a desktop compositor;
- keep multiple reference-distro implementations active without an explicit decision and independent maintenance lifecycle.
