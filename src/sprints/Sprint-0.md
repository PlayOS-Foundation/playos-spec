# Sprint 0 — Build and UEFI Foundation

**Goal:** Establish the six-repository structure and a reproducible Buildroot factory that produces a minimal PlayOS EFI image bootable through QEMU/OVMF.

**Primary Outcome:** Any developer can clone the repos, run `make qemu-build && make qemu-run`, and see the system boot to a BusyBox shell through UEFI in QEMU.

**Prerequisites:** None — this is the first sprint.

---

## Key Deliverables

### Repository Structure
- Create six GitHub repositories: `playos-spec`, `playos-platform-api`, `playos-runtime`, `playos-compositor`, `playos-shell`, `playos-refdistro`
- Add top-level `README.md` to each with: purpose, dependency position, build instructions
- Add `CONTRIBUTING.md` and branch protection rules to each

### `playos-refdistro` Build System
- Add official Buildroot as a pinned Git submodule under `buildroot/`
- Create `br2-external/` tree with:
  - `external.desc`, `Config.in`, `external.mk`
  - `configs/playos_qemu_x86_64_defconfig` — minimal x86_64 UEFI config
  - `board/playos/common/` and `board/playos/qemu-x86_64/`
  - Stub package directories: `playos-init/`, `playos-platform-api/`, `playos-runtime/`, `playos-compositor/`, `playos-shell/`
- `versions.lock` — pins Buildroot commit, Linux version, toolchain, all component repos
- `Makefile` with stable developer commands:
  ```
  make setup        # install host dependencies
  make qemu-config  # configure Buildroot for QEMU target
  make qemu-build   # full Buildroot build
  make qemu-run     # launch QEMU/OVMF with built image
  make ally-config  # configure for ROG Ally (stub, becomes real in Sprint 3)
  make ally-build   # ROG Ally build (stub)
  make clean        # clean build output
  ```

### Kernel Configuration
- Start from a known-working upstream Linux LTS config for x86_64
- Enable: EFI stub, initramfs embedding, devtmpfs, procfs, sysfs, tmpfs, serial console, virtio for QEMU
- Disable: unneeded drivers, server features, virtualization host stack
- Produce kernel with embedded initramfs

### Minimal Initramfs
- BusyBox-based rootfs for this sprint
- `/init` stub script: mount virtual FSes, print boot message, drop to BusyBox `sh`
- No `playos-init` yet — that is Sprint 1

### UEFI Boot Artifact
- Build script or Buildroot post-image hook that:
  - Wraps kernel + initramfs as a single UEFI application using the Linux EFI stub
  - Produces `playos-esp.img` with `/EFI/BOOT/BOOTX64.EFI`
- QEMU run command boots via OVMF (not `-kernel` shortcut) to validate the real UEFI path

### Toolchain
- GCC cross-compiler targeting `x86_64-linux-musl`
- musl libc only — no glibc
- Validate that a trivial C program compiles and runs in the initramfs

### CI (GitHub Actions)
- Trigger: push to any repo's `main` or PR branch
- Steps: `make setup`, `make qemu-build`, boot test via QEMU with timeout assertion

---

## Acceptance Criteria

- [ ] `make setup && make qemu-build` succeeds from a clean checkout
- [ ] QEMU boots the EFI image through OVMF (not `-kernel`)
- [ ] `/init` runs; a shell prompt appears in the QEMU serial output
- [ ] Kernel reports correct x86_64 musl userspace
- [ ] `versions.lock` is present and all entries are pinned to full commits
- [ ] CI passes on `main`
- [ ] All six repositories exist with README files

---

## Repositories Primarily Involved

| Repo | Work |
|---|---|
| `playos-refdistro` | All build system, Buildroot config, boot artifact, Makefile, CI |
| `playos-spec` | Initial architecture doc, ADR-0001 (repo structure decision) |
| All others | Create repo + README stub only |

---

## Testing Approach

- Automated: `make qemu-run` with QEMU exit-on-panic and a serial-output grep for the shell prompt
- Manual: developer runs `make qemu-run` and types a BusyBox command
- No physical hardware testing this sprint

---

## Exit Gate

A clean `make qemu-build && make qemu-run` on a fresh CI runner produces a UEFI-bootable image that reaches a BusyBox shell in QEMU/OVMF.

*Next: [Sprint 1 — `playos-init` and Minimal Boot Supervision](Sprint-1.md)*
