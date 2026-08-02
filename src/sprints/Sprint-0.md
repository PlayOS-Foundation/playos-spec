# Sprint 0 — Build and UEFI Foundation

**Goal:** Establish the repository layout, Buildroot integration, and the first reproducible QEMU/OVMF boot path for PlayOS.

**Primary Outcome:** A clean environment can build and boot a minimal EFI image through QEMU/OVMF and reach a BusyBox shell using the PlayOS reference distribution workflow.

**Prerequisites:** None — this is the first sprint.

---

## Why This Sprint Exists

Sprint 0 is the factory bootstrap sprint. It creates the build environment, repo boundaries, version pinning rules, and boot artifact shape that every later sprint depends on.

Without this sprint being strict and reproducible, later work becomes impossible to compare, debug, or automate.

---

## Start Condition Checklist

- A Linux build environment is available (native Linux, WSL2, or Linux VM).
- QEMU and OVMF are available or installable.
- The six PlayOS repositories either already exist or will be created as part of the sprint.
- The implementation agent is allowed to create build/config scaffolding across repositories.

---

## Decisions Locked for This Sprint

- **Distribution assembler repo:** `playos-refdistro`
- **Documentation repo:** `playos-spec`
- **C library/toolchain policy:** musl only
- **Boot validation path:** UEFI through OVMF only; do not use the `-kernel` shortcut as the main proof
- **Initial userspace:** BusyBox-based initramfs for bootstrap only
- **Kernel target for this sprint:** QEMU x86_64, not ROG Ally hardware yet
- **Target build host OS:** Ubuntu Server (LTS) — all host setup scripts and CI assume Ubuntu
- **Shell logging framework:** a shared bash library (`scripts/lib/playos_log.sh`) is created here and used by every script in the project from this sprint forward

---

## Scope

### In Scope

- repository bootstrap and baseline documentation files
- Buildroot integration strategy
- `br2-external` skeleton
- QEMU x86_64 defconfig
- minimal kernel/initramfs boot path
- UEFI boot artifact generation
- version pinning conventions
- **Ubuntu Server host environment setup script**
- **shared bash logging framework for all project scripts**
- first CI shape for image build and boot proof

### Explicitly Out of Scope

- real `playos-init`
- compositor or Wayland logic
- physical ROG Ally boot
- real shell UI
- storage, audio, power, or update systems

---

## Required Repository Changes

| Repo | Required work |
|---|---|
| `playos-spec` | initial project docs and ADRs that define repo boundaries |
| `playos-refdistro` | Buildroot tree, defconfig, image generation, make targets, CI |
| `playos-platform-api` | repo scaffold only |
| `playos-runtime` | repo scaffold only |
| `playos-compositor` | repo scaffold only |
| `playos-shell` | repo scaffold only |

---

## Expected Files and Directories

### `playos-refdistro`

```text
Makefile
versions.lock
buildroot/                     # pinned Buildroot checkout or submodule

scripts/
├── lib/
│   └── playos_log.sh          # shared bash logging framework (used by ALL scripts)
├── setup-ubuntu.sh            # Ubuntu Server host environment bootstrap
└── qemu-boot-check.sh         # CI boot assertion script

br2-external/
├── external.desc
├── Config.in
├── external.mk
├── configs/
│   └── playos_qemu_x86_64_defconfig
├── board/
│   ├── common/
│   └── qemu-x86_64/
└── package/
    ├── playos-init/
    ├── playos-platform-api/
    ├── playos-runtime/
    ├── playos-compositor/
    └── playos-shell/

.github/workflows/
└── qemu-build.yml
```

### All repositories

```text
README.md
CONTRIBUTING.md
AGENTS.md
.gitignore
```

---

## Agent Task Breakdown

### Task Status Grid

Update the **Status** column as work progresses: `not started` → `in progress` → `blocked` or `done`.

| Task ID | Task | Primary repo | Status | Notes / evidence |
|---|---|---|---|---|
| S0-T1 | Create or validate the six-repository structure | cross-repo | not started | |
| S0-T2 | Add the Buildroot integration skeleton | `playos-refdistro` | not started | |
| S0-T3 | Create the QEMU x86_64 defconfig | `playos-refdistro` | not started | |
| S0-T4 | Build the minimal kernel + initramfs path | `playos-refdistro` | not started | |
| S0-T5 | Produce the real UEFI boot artifact | `playos-refdistro` | not started | |
| S0-T6 | Standardise developer commands | `playos-refdistro` | not started | |
| S0-T7 | Create and enforce version pinning | `playos-refdistro` | not started | |
| S0-T8 | Add first-pass CI | `playos-refdistro` | not started | |
| S0-T9 | Create Ubuntu Server host environment setup script | `playos-refdistro` | not started | |
| S0-T10 | Create shared bash logging framework | `playos-refdistro` | not started | |

### S0-T1 — Create or validate the six-repository structure

- Ensure the following repositories exist:
  - `playos-spec`
  - `playos-platform-api`
  - `playos-runtime`
  - `playos-compositor`
  - `playos-shell`
  - `playos-refdistro`

- Add or validate baseline repo files:
  - `README.md`
  - `CONTRIBUTING.md`
  - `AGENTS.md`
  - `.gitignore`

**Done when:** the workspace has the six clean repo boundaries that later sprints can target explicitly.

### S0-T2 — Add the Buildroot integration skeleton

- Bring in upstream Buildroot as a pinned checkout/submodule under `playos-refdistro\buildroot\`.
- Create the `br2-external` tree.
- Add top-level `Config.in`, `external.mk`, and `external.desc`.
- Add stub package directories for every component repo.

**Done when:** Buildroot can see the PlayOS `br2-external` tree and package stubs.

### S0-T3 — Create the QEMU x86_64 defconfig

- Add `playos_qemu_x86_64_defconfig`.
- Base it on a minimal EFI-capable x86_64 configuration.
- Ensure these capabilities are included:
  - EFI boot
  - initramfs support
  - devtmpfs/procfs/sysfs/tmpfs
  - serial console
  - virtio devices needed by QEMU

**Done when:** the image can be configured reproducibly for the QEMU path.

### S0-T4 — Build the minimal kernel + initramfs path

- Use a BusyBox-based initramfs.
- Provide an `/init` script that:
  1. mounts virtual filesystems
  2. prints a clear boot banner
  3. drops to a BusyBox shell

- Do not implement the real `playos-init` here. That is Sprint 1.

**Done when:** QEMU reaches the BusyBox shell through the PlayOS image.

### S0-T5 — Produce the real UEFI boot artifact

- Generate an EFI-bootable image with:

```text
/EFI/BOOT/BOOTX64.EFI
```

- Use OVMF to boot it.
- Avoid treating `qemu -kernel` as equivalent proof.

**Done when:** the image boots through the same UEFI path expected for real devices.

### S0-T6 — Standardise developer commands

The `Makefile` must expose a stable developer interface:

```text
make setup
make qemu-config
make qemu-build
make qemu-run
make ally-config
make ally-build
make clean
```

- `ally-*` targets may be stubs in this sprint, but they must exist and document that Sprint 3 makes them real.

**Done when:** a new developer has one predictable command surface.

### S0-T7 — Create and enforce version pinning

- Add `versions.lock`.
- Pin Buildroot, Linux, toolchain assumptions, and every PlayOS component reference in a documented format.
- Use full commit SHAs, not floating branch names.

**Done when:** the sprint output is reproducible and reviewable.

### S0-T8 — Add first-pass CI

- Add a GitHub Actions workflow or equivalent CI definition in `playos-refdistro`.
- Build the QEMU image.
- Boot QEMU with a timeout.
- Assert success by matching serial output from `/init`.
- Use `scripts/qemu-boot-check.sh` (created in S0-T9) for the boot assertion step.
- Use `scripts/lib/playos_log.sh` for all CI script output.

**Done when:** clean CI can prove the boot path automatically.

### S0-T9 — Create Ubuntu Server host environment setup script

Create `scripts/setup-ubuntu.sh` — the single authoritative script that prepares a fresh Ubuntu Server LTS machine for PlayOS development and CI.

**The script must:**

- detect Ubuntu version and fail clearly if unsupported (minimum: Ubuntu 22.04 LTS)
- run non-interactively (`-y` flags, no manual prompts)
- install all packages needed for Buildroot builds:

```bash
# Core build tools
build-essential gcc g++ make cmake ninja-build
# Buildroot host dependencies
libncurses-dev libssl-dev libelf-dev bison flex
# Image and boot tooling
ovmf qemu-system-x86 qemu-utils dosfstools mtools parted
# Filesystem and EFI tools
gdisk squashfs-tools
# Python (for Buildroot scripts)
python3 python3-pip
# Git and versioning
git git-lfs curl wget
# Testing and introspection tools
evtest alsa-utils pciutils usbutils
```

- validate each critical tool is present after install (e.g. `which qemu-system-x86_64`, `ovmf` package presence)
- print a clear summary: what was installed, what was already present, what failed
- use `scripts/lib/playos_log.sh` for all output
- be idempotent — safe to run on a machine that already has everything installed

**Usage:**

```bash
bash scripts/setup-ubuntu.sh
```

**Done when:** a completely fresh Ubuntu Server 22.04 LTS machine can run `bash scripts/setup-ubuntu.sh` and then immediately run `make qemu-build` with no missing dependency errors.

### S0-T10 — Create shared bash logging framework

Create `scripts/lib/playos_log.sh` — a small, dependency-free bash library sourced by every PlayOS script in every repository.

**Requirements:**

- one-liner source contract: `. "$(dirname "$0")/../lib/playos_log.sh"` or equivalent relative path
- six log level functions with timestamped, coloured, and level-tagged output:

```bash
playos_log_debug "tag" "message"   # grey,  [DEBUG]
playos_log_info  "tag" "message"   # white, [INFO]
playos_log_ok    "tag" "message"   # green, [OK]
playos_log_warn  "tag" "message"   # yellow,[WARN]
playos_log_error "tag" "message"   # red,   [ERROR]
playos_log_fatal "tag" "message"   # red bold, [FATAL] — also calls exit 1
```

- output format: `[2026-08-02 19:55:03] [LEVEL] [TAG] message`
- colour is applied when stdout is a terminal (`-t 1`); stripped when piped to a file or CI log
- `PLAYOS_LOG_LEVEL` environment variable controls minimum visible level (default: `INFO`)
- a `playos_log_step` helper for section banners:

```bash
playos_log_step "Running Buildroot configuration"
# prints:
# ─────────────────────────────────────────────
#  ▶  Running Buildroot configuration
# ─────────────────────────────────────────────
```

- zero external dependencies — pure bash, no Python, no `jq`, no colour libraries

**Contract rules for all project scripts:**

- every script in `playos-refdistro/scripts/` must source `playos_log.sh`
- `echo` is not used for informational output in any PlayOS script; use `playos_log_*` instead
- CI step names in `.github/workflows/` use `playos_log_step` to mark phases

**Done when:** `setup-ubuntu.sh`, `Makefile`, and CI scripts all source and use the library; output is consistently formatted across all entry points.

---

## Implementation Guidance

### Buildroot structure

- Keep PlayOS-specific logic in `br2-external`, not as random patches scattered across Buildroot.
- Separate board-common and QEMU-specific files so Sprint 3 can later add Ally-specific files cleanly.

### Toolchain policy

- Use musl from the start so later ABI and runtime decisions do not need to be reworked.
- Validate the toolchain by compiling and running a trivial C program inside the initramfs.

### Reproducibility

- Avoid hidden local prerequisites in scripts.
- Every required host package must be installed through `scripts/setup-ubuntu.sh`, not documented only in a README.
- The setup script is the only authoritative source for what the build host needs.

### Ubuntu Server as the build host

- All CI runners and developer machines are expected to run Ubuntu Server 22.04 LTS or later.
- `scripts/setup-ubuntu.sh` must be the first command run on a fresh machine.
- Never assume a package is present unless it is explicitly installed by that script.

### Logging conventions

- All shell scripts source `scripts/lib/playos_log.sh`.
- No script uses bare `echo` for informational output.
- This convention applies to Sprint 0 scripts and must be maintained by all later sprints.

---

## Verification and Evidence

| Evidence | How it is produced |
|---|---|
| Repo proof | directory listing or project inventory of all six repos |
| Buildroot proof | `br2-external` tree present and referenced by the build |
| Boot proof | serial log showing EFI boot and BusyBox shell |
| UEFI proof | OVMF boot path, not direct kernel boot |
| Reproducibility proof | `bash scripts/setup-ubuntu.sh && make qemu-build` from a clean Ubuntu Server 22.04 |
| Versioning proof | `versions.lock` populated with pinned values |
| Setup proof | `setup-ubuntu.sh` runs to completion on a fresh machine with a clean summary |
| Logging proof | all sprint scripts emit structured `playos_log_*` output, not bare `echo` |

---

## Acceptance Criteria

- [ ] all six repositories exist with baseline repo files
- [ ] `playos-refdistro` contains a valid `br2-external` skeleton
- [ ] `playos_qemu_x86_64_defconfig` exists
- [ ] a BusyBox initramfs boots through OVMF in QEMU
- [ ] `/init` mounts the expected virtual filesystems and reaches a shell
- [ ] the developer `Makefile` exposes the standard command surface
- [ ] `versions.lock` exists and uses pinned values
- [ ] CI can build and boot-check the image automatically
- [ ] `scripts/setup-ubuntu.sh` prepares a fresh Ubuntu Server 22.04 LTS machine end-to-end
- [ ] `scripts/setup-ubuntu.sh` is idempotent and validates all installed tools
- [ ] `scripts/lib/playos_log.sh` exists and is sourced by all project scripts
- [ ] all scripts emit `playos_log_*` output — no bare `echo` for informational messages
- [ ] the sprint can be reproduced from a clean Ubuntu Server environment using only the setup script

---

## Handoff to Sprint 1

Sprint 1 may assume:

- the repo layout is stable
- the Buildroot path is real and reproducible
- a QEMU/OVMF boot loop already exists
- `/init` may now be replaced by the real `playos-init`
- `scripts/setup-ubuntu.sh` is the authoritative way to prepare a build host
- `scripts/lib/playos_log.sh` exists and all new scripts must source it

Sprint 1 should build on this boot foundation rather than changing the factory shape unless a documented blocker requires it.

---

## Exit Gate

A clean `make qemu-build && make qemu-run` on a fresh environment produces a UEFI-bootable PlayOS image that reaches a BusyBox shell in QEMU/OVMF.

*Next: [Sprint 1 — `playos-init` and Minimal Boot Supervision](Sprint-1.md)*
