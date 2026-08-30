# Sprint 3 — ROG Ally Kernel and Device Bring-Up

**Goal:** Boot PlayOS from USB on physical ROG Ally hardware with all essential devices working: display, controller input, audio, storage, battery, and thermal reporting. Define the first hardware-backed `libplayos` input contract and prototype backend.

**Primary Outcome:** PlayOS boots from removable media on a real ROG Ally, enumerates the essential devices needed for the console lifecycle, and provides a compiling prototype of the public input API backed by evdev.

**Prerequisites:** Sprint 2 complete — the compositor skeleton works in QEMU/nested mode and `playos-init` supervision is stable.

---

## Why This Sprint Exists

Sprint 3 is the hardware qualification sprint. Up to this point the project proves architecture and runtime shape. This sprint proves the software stack can identify and use the actual Ally hardware without unsafe assumptions.

---

## Start Condition Checklist

- Sprint 2 software path still boots in QEMU.
- A physical ASUS ROG Ally is available for testing.
- A USB boot workflow already exists from Sprint 0.
- A Linux host environment is available to produce images and inspect logs.

---

## Decisions Locked for This Sprint

- **Canonical defconfig name:** `br2-external\configs\playos_ally_defconfig`
- **Public input language surface:** C99 public headers in `playos-platform-api`
- **Input backend strategy:** evdev prototype only for this sprint
- **Button representation:** **bitmask flags**, not enum indices
- **Reserved buttons:** `PLAYOS_BUTTON_SYSTEM` and `PLAYOS_BUTTON_QUICK_MENU` are defined publicly but must not be delivered to games
- **No game-facing audio/power API yet:** hardware may be verified, but those public APIs belong to later sprints

---

## Scope

### In Scope

- Ally-specific kernel configuration
- USB-bootable image for the Ally
- firmware inclusion needed for AMDGPU and platform support
- verification scripts for the essential devices
- public input header finalisation for the first API group
- evdev prototype backend for input
- physical hardware testing and evidence capture

### Explicitly Out of Scope

- polished shell UX
- native DRM/KMS compositor ownership of the display
- full audio API design
- suspend/resume behaviour
- installer or internal SSD deployment

---

## Required Repository Changes

| Repo | Required work |
|---|---|
| `playos-refdistro` | Ally defconfig, firmware packaging, USB image target, device verification tooling |
| `playos-platform-api` | Public input header contract and evdev prototype backend |
| `playos-spec` | Input mapping reference and any clarified hardware notes |

---

## Expected Files and Directories

### `playos-refdistro`

```text
br2-external/configs/
└── playos_ally_defconfig

tools/hw-check/
├── check-display.sh
├── check-input.sh
├── check-audio.sh
├── check-storage.sh
├── check-power.sh
└── run-all.sh
```

### `playos-platform-api`

```text
include/playos/
└── playos_input.h

src/
└── playos_input_evdev.c

docs/
└── rog-ally-input-mapping.md

tests/
└── input/
```

---

## Agent Task Breakdown

### Task Status Grid

Update the **Status** column as work progresses: `not started` → `in progress` → `blocked` or `done`.

| Task ID | Task | Primary repo | Status | Notes / evidence |
|---|---|---|---|---|
| S3-T1 | Create the Ally defconfig and boot image path | `playos-refdistro` | done | `playos_ally_defconfig`, Makefile targets |
| S3-T2 | Enable the required kernel subsystems | `playos-refdistro` | done | `board/ally/linux.config`, EFI stub, AMDGPU, all subsystems |
| S3-T3 | Package required firmware | `playos-refdistro` | done | AMDGPU blobs + AMD ucode via linux-firmware |
| S3-T4 | Add device verification tooling | `playos-refdistro` | done | `tools/hw-check/` (6 scripts), all PASSED on Ally |
| S3-T5 | Finalise the public input contract | `playos-platform-api` | done | `playos_input.h` with bitmask buttons |
| S3-T6 | Implement the evdev prototype backend | `playos-platform-api` | done | `src/backends/backend_evdev.c`, auto-discovery |
| S3-T7 | Document the hardware mapping | `playos-platform-api`, `playos-spec` | done | `docs/rog-ally-input-mapping.md` |
| S3-T8 | Capture physical hardware evidence | `playos-refdistro` | done | Ally booted from USB, all hw-check tests PASSED |

### S3-T1 — Create the Ally defconfig and boot image path

- Create `playos_ally_defconfig`.
- Start from a known-good Ally-capable Linux configuration, then trim conservatively.
- Add a `make ally-usb-image` target that produces a removable-media boot artifact.
- Keep the existing QEMU config intact.

**Done when:** the image is buildable and intended specifically for the Ally path.

### S3-T2 — Enable the required kernel subsystems

The defconfig must include at least the following classes of support:

| Subsystem | Required symbols or equivalent |
|---|---|
| UEFI and x86_64 | `CONFIG_EFI_STUB`, `CONFIG_ACPI`, `CONFIG_X86_64` |
| PCIe and IOMMU | `CONFIG_PCI`, `CONFIG_AMD_IOMMU` |
| Virtual filesystems | `devtmpfs`, `procfs`, `sysfs`, `tmpfs` |
| Serial console | `CONFIG_SERIAL_8250_CONSOLE` or equivalent |
| DRM/KMS and AMDGPU | `CONFIG_DRM`, `CONFIG_DRM_AMDGPU`, `CONFIG_DRM_AMD_DC` |
| Recovery graphics | `CONFIG_DRM_SIMPLEDRM` |
| USB xHCI | `CONFIG_USB_XHCI_HCD` |
| Input | `CONFIG_HID`, `CONFIG_INPUT_EVDEV`, `CONFIG_HID_ASUS` |
| Audio | `CONFIG_SND_HDA_INTEL`, `CONFIG_SND_SOC`, AMD ACP support |
| Storage | `CONFIG_BLK_DEV_NVME` |
| Filesystems | `CONFIG_FAT_FS`, `CONFIG_EXT4_FS` |
| Power and thermal | `CONFIG_THERMAL`, `CONFIG_BATTERY_ACPI`, `CONFIG_X86_AMD_PSTATE` |
| Watchdog | `CONFIG_WATCHDOG` |

**Done when:** the image boots and all essential device classes enumerate.

### S3-T3 — Package required firmware

- Include AMDGPU firmware blobs.
- Include AMD CPU microcode if required by the chosen boot path.
- Include any Ally-specific firmware needed by the selected kernel/drivers.
- Document firmware source expectations for reproducible builds.

**Done when:** the GPU and essential platform devices initialise without missing-firmware failures.

### S3-T4 — Add device verification tooling

For each device class, add one simple verifiable check:

| Device | Verification target |
|---|---|
| Display | `/dev/dri/card*` exists and can be inspected |
| Render node | `/dev/dri/renderD*` exists |
| Controller | `/dev/input/event*` exists and emits expected button/stick activity |
| Audio | `aplay -l` and a short playback check succeed |
| NVMe | block device exists and partitions are readable |
| Battery | `/sys/class/power_supply/` exposes battery and AC state |
| Thermal | `/sys/class/thermal/` exposes thermal zones |

- Make the combined output land in `/run/playos/hw-check.log`.

**Done when:** one command or script can produce a single hardware bring-up report.

### S3-T5 — Finalise the public input contract

The sprint must settle the first public input ABI in `include/playos/playos_input.h`.

Use **bitmask values** for buttons:

```c
typedef uint32_t playos_button_mask_t;

enum {
    PLAYOS_BUTTON_SOUTH      = 1u << 0,
    PLAYOS_BUTTON_EAST       = 1u << 1,
    PLAYOS_BUTTON_WEST       = 1u << 2,
    PLAYOS_BUTTON_NORTH      = 1u << 3,
    PLAYOS_BUTTON_START      = 1u << 4,
    PLAYOS_BUTTON_SELECT     = 1u << 5,
    PLAYOS_BUTTON_SYSTEM     = 1u << 6,
    PLAYOS_BUTTON_QUICK_MENU = 1u << 7,
    PLAYOS_BUTTON_DPAD_UP    = 1u << 8,
    PLAYOS_BUTTON_DPAD_DOWN  = 1u << 9,
    PLAYOS_BUTTON_DPAD_LEFT  = 1u << 10,
    PLAYOS_BUTTON_DPAD_RIGHT = 1u << 11,
    PLAYOS_BUTTON_L1         = 1u << 12,
    PLAYOS_BUTTON_R1         = 1u << 13,
    PLAYOS_BUTTON_L3         = 1u << 14,
    PLAYOS_BUTTON_R3         = 1u << 15
};

typedef enum {
    PLAYOS_AXIS_LEFT_X = 0,
    PLAYOS_AXIS_LEFT_Y,
    PLAYOS_AXIS_RIGHT_X,
    PLAYOS_AXIS_RIGHT_Y,
    PLAYOS_AXIS_LEFT_TRIGGER,
    PLAYOS_AXIS_RIGHT_TRIGGER,
    PLAYOS_AXIS_COUNT
} playos_axis_t;

typedef struct {
    playos_button_mask_t buttons;
    float axes[PLAYOS_AXIS_COUNT];
} playos_controller_state_t;
```

Function expectations:

- `playos_input_controller_connected()`
- `playos_input_get_controller_state()`

**Rule:** `PLAYOS_BUTTON_SYSTEM` and `PLAYOS_BUTTON_QUICK_MENU` are reserved identifiers. The backend may observe them on hardware, but game-facing snapshots must not report them once compositor interception exists.

**Done when:** the header compiles cleanly and the contract is specific enough for shell/game consumers.

### S3-T6 — Implement the evdev prototype backend

- Identify the Ally controller event node(s).
- Map physical event codes to the public logical button and axis contract.
- Normalize sticks to `[-1.0, 1.0]`.
- Normalize triggers consistently and document whether they use `[0.0, 1.0]`.
- Ignore or reserve unsupported extra buttons for now, but document them in the mapping file.

**Done when:** a small test program can poll and print logical controller state on the Ally.

### S3-T7 — Document the hardware mapping

- Add `docs/rog-ally-input-mapping.md`.
- Record Linux event codes, axis ranges, and any quirks.
- Mark which controls are public in MVP and which are deferred.

**Done when:** future shell/game work does not need to rediscover controller details experimentally.

### S3-T8 — Capture physical hardware evidence

- Record boot success from USB.
- Record hardware check output.
- Record input test output with at least A/B/X/Y, D-pad, sticks, and triggers.
- Record any missing devices or kernel warnings.

**Done when:** the sprint leaves behind a reproducible bring-up record instead of memory-only claims.

---

## Implementation Guidance

### Defconfig naming and consistency

Use `playos_ally_defconfig` everywhere in docs, scripts, and Make targets. Do not introduce `playos_rog_ally_defconfig` as a second name.

### Input contract stability

- Buttons are flags because multiple buttons may be pressed simultaneously.
- Axes are array-indexed because they are numeric channels, not bitfields.
- Avoid exposing kernel event codes directly in the public header.

### Verification philosophy

This sprint is complete only when device presence is tied to a concrete script or command. "It seemed to work once" is not enough evidence.

---

## Acceptance Criteria

- [x] `playos_ally_defconfig` exists and is used by the Ally build path
- [x] a USB image can be produced for the Ally
- [x] the Ally boots PlayOS from removable media
- [x] AMDGPU and DRM device nodes appear
- [x] the controller appears through evdev and emits expected events
- [x] audio hardware is visible and can play a short test sample
- [x] NVMe, battery, and thermal information are visible
- [x] `/run/playos/hw-check.log` is produced by the verification tooling
- [x] `playos_input.h` defines a stable public input contract
- [x] button values are represented as bitmask flags
- [x] the evdev backend prototype reads controller state on the Ally
- [x] the hardware input mapping document is committed

---

## Handoff to Sprint 4

Sprint 4 may assume:

- the Ally kernel and firmware stack can boot reliably
- the AMD GPU and DRM device nodes exist
- the public input ABI shape is known
- physical-hardware verification scripts already exist

Sprint 4 should consume this hardware baseline and focus on native compositor ownership of the display.

---

## Exit Gate

A PlayOS USB image boots on physical ROG Ally hardware, essential devices enumerate correctly, and the first hardware-backed `libplayos` input API contract and evdev prototype are in place.

*Previous: [Sprint 2.5](Sprint-2.5.md) | Next: [Sprint 4](Sprint-4.md)*

---

## Sprint 3 Outcomes

**Status: COMPLETE** — all 8 tasks done, committed, and verified on physical ROG Ally hardware.

### Deliverables

| Repo | Commits | Key Artifacts |
|---|---|---|
| `playos-refdistro` | `9305481`, `d31ec48`, `561b701`, `b6fbb69`, `e9b0c44`, `65117f2` | Ally defconfig, kernel config, USB image script, flash script, hw-check tools |
| `playos-platform-api` | `d7a0050`, `580026c` | Input header, evdev backend, input mapping docs, API stubs |

### Key Technical Decisions

1. **EFI stub boot, not GRUB** — kernel bzImage with embedded initramfs placed directly as `EFI/BOOT/BOOTX64.EFI`. UEFI firmware boots the kernel without an intermediate bootloader. Simpler, faster, fewer dependencies.

2. **Embedded initramfs** — `BR2_LINUX_KERNEL_INITRAMFS_SOURCE` is critical. Without it the kernel panics because it has no rootfs. The 178MB cpio gzips to ~59MB inside the bzImage.

3. **No modules** — all kernel drivers built-in (`# CONFIG_MODULES is not set`). Simplifies the boot path — no module loading, no initramfs module discovery.

4. **BusyBox retained for debugging** — production should strip it, but kept for Sprint 3 hardware verification (need a shell to run hw-check).

### Lessons Learned

1. **`lsblk` columns break on model names with spaces** — "SanDisk 3.2Gen1" gets split into two columns. Use `lsblk -P` (key=value pairs) for reliable parsing.

2. **GPT backup header consumes disk space** — partition sizes must account for ~34 sectors at end. Use `sgdisk -n N:0:0` (fill remaining) for the last partition instead of fixed size.

3. **Buildroot `BR2_LINUX_KERNEL_INITRAMFS_SOURCE` is easy to miss** — the kernel compiles fine without it but panics at boot. Consider adding a post-build check that verifies initramfs is embedded.

4. **Ally boots reliably from USB via Volume Down + Power** — no Secure Boot key enrollment needed (the Ally's UEFI has Secure Boot disabled by default).

5. **SP5100 is the watchdog chip on ROG Ally** — needs `CONFIG_SP5100_TCO`, not generic iTCO.

6. **Kernel cmdline fallback matters** — `CONFIG_CMDLINE="console=tty1 quiet loglevel=3"` ensures boot works even when UEFI doesn't supply cmdline.
