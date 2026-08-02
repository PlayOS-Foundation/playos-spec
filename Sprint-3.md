# Sprint 3 — ROG Ally Kernel and Device Bring-Up

**Goal:** Boot PlayOS from USB on physical ROG Ally hardware with all essential devices working: display (AMDGPU), controller input (HID/evdev), audio (ALSA), and storage (NVMe). Define the first qualified Platform API input backend contract.

**Primary Outcome:** PlayOS boots from USB on a ROG Ally. The BusyBox/diagnostic shell is reachable. All critical devices are enumerated and functional. The `libplayos` input API contract is defined and a prototype backend compiles.

**Prerequisites:** Sprint 2 complete — `playos-compositor` skeleton working in QEMU.

---

## Key Deliverables

### ROG Ally Kernel Configuration

Create `br2-external/configs/playos_rog_ally_defconfig`. Start from a known-working ROG Ally Linux configuration (e.g., Arch Linux on Ally or ChimeraOS kernel config) and trim conservatively.

**Required subsystems (all must be built-in or as modules):**

| Subsystem | Config symbols |
|---|---|
| x86_64, EFI stub, ACPI | `CONFIG_EFI_STUB`, `CONFIG_ACPI`, `CONFIG_X86_64` |
| PCIe, IOMMU | `CONFIG_PCI`, `CONFIG_AMD_IOMMU` |
| devtmpfs, procfs, sysfs, tmpfs, initramfs | standard |
| Serial/UART console | `CONFIG_SERIAL_8250_CONSOLE` or equivalent |
| DRM/KMS + AMDGPU | `CONFIG_DRM`, `CONFIG_DRM_AMDGPU`, `CONFIG_DRM_AMD_DC` |
| SimpleDRM | `CONFIG_DRM_SIMPLEDRM` (recovery fallback) |
| USB xHCI | `CONFIG_USB_XHCI_HCD` |
| HID, evdev | `CONFIG_HID`, `CONFIG_INPUT_EVDEV` |
| ASUS HID quirks | `CONFIG_HID_ASUS` |
| ALSA HDA/SoC/ACP | `CONFIG_SND_HDA_INTEL`, `CONFIG_SND_SOC`, AMD ACP driver |
| NVMe | `CONFIG_BLK_DEV_NVME` |
| FAT32, ext4 | `CONFIG_FAT_FS`, `CONFIG_EXT4_FS` |
| Thermal, ACPI battery, AMD P-state | `CONFIG_THERMAL`, `CONFIG_BATTERY_ACPI`, `CONFIG_X86_AMD_PSTATE` |
| Watchdog | `CONFIG_WATCHDOG` |

**Required firmware (embed into initramfs or load from `/lib/firmware`):**
- `amdgpu/` — ROG Ally GPU firmware blobs
- `amd-ucode/` — CPU microcode
- Any ASUS HID firmware needed

**USB boot:**
- Build a USB-bootable EFI image (same structure as QEMU image: `/EFI/BOOT/BOOTX64.EFI`)
- Add `make ally-usb-image` target to the `Makefile`
- Write to USB via `dd` or a helper script

### Device Bring-Up Verification

For each device, create a simple verification script or test binary:

| Device | Verification |
|---|---|
| Display | `/dev/dri/card*` exists; `modetest` shows connectors and modes |
| GPU render node | `/dev/dri/renderD*` exists |
| Controller | `/dev/input/event*` exists; `evtest` shows button events |
| Audio | `aplay -l` shows capture/playback devices; test PCM with `speaker-test` |
| NVMe | `/dev/nvme0` exists; partition table readable |
| Battery | `/sys/class/power_supply/` shows battery and adapter |
| Thermal | `/sys/class/thermal/` shows thermal zones |

### Platform API Input Contract — `playos-platform-api`

Define the stable C ABI for input. This is the first `libplayos` API group to be formally specified.

**Header: `include/playos/playos_input.h`**

```c
typedef enum {
    PLAYOS_BUTTON_SOUTH       = 0,
    PLAYOS_BUTTON_EAST        = 1,
    PLAYOS_BUTTON_WEST        = 2,
    PLAYOS_BUTTON_NORTH       = 3,
    PLAYOS_BUTTON_START       = 4,
    PLAYOS_BUTTON_SELECT      = 5,
    PLAYOS_BUTTON_SYSTEM      = 6,   /* reserved — never delivered to games */
    PLAYOS_BUTTON_QUICK_MENU  = 7,   /* reserved */
    PLAYOS_BUTTON_DPAD_UP     = 8,
    PLAYOS_BUTTON_DPAD_DOWN   = 9,
    PLAYOS_BUTTON_DPAD_LEFT   = 10,
    PLAYOS_BUTTON_DPAD_RIGHT  = 11,
    PLAYOS_BUTTON_L1          = 12,
    PLAYOS_BUTTON_R1          = 13,
    PLAYOS_BUTTON_L3          = 14,
    PLAYOS_BUTTON_R3          = 15,
    PLAYOS_BUTTON_COUNT       = 16
} PlayOSButton;

typedef enum {
    PLAYOS_AXIS_LEFT_X        = 0,
    PLAYOS_AXIS_LEFT_Y        = 1,
    PLAYOS_AXIS_RIGHT_X       = 2,
    PLAYOS_AXIS_RIGHT_Y       = 3,
    PLAYOS_AXIS_LEFT_TRIGGER  = 4,
    PLAYOS_AXIS_RIGHT_TRIGGER = 5,
    PLAYOS_AXIS_COUNT         = 6
} PlayOSAxis;

typedef struct {
    uint32_t buttons;       /* bitmask of PlayOSButton values */
    float    axes[PLAYOS_AXIS_COUNT]; /* normalized [-1.0, 1.0] or [0.0, 1.0] for triggers */
} PlayOSControllerState;

/* Returns 1 if the platform controller is connected, 0 otherwise. */
int playos_input_controller_connected(void);

/* Fills state with the current controller snapshot. Returns 0 on success. */
int playos_input_get_controller_state(PlayOSControllerState *state);
```

**Backend:** Implement a prototype evdev backend that reads from `/dev/input/event*`, maps ROG Ally button/axis codes to the logical constants, and fills `PlayOSControllerState`.

**ROG Ally button mapping document:** Create `playos-platform-api/docs/rog-ally-input-mapping.md` — maps physical event codes to logical PlayOS constants.

---

## Acceptance Criteria

- [ ] ROG Ally boots from USB to a BusyBox/diagnostic shell
- [ ] AMDGPU driver loads: `/dev/dri/card*` and `/dev/dri/renderD*` present
- [ ] `modetest` shows the Ally's display connector with correct resolution
- [ ] ROG Ally controller visible in `/dev/input/event*`; `evtest` shows A/B/X/Y and stick events
- [ ] `aplay -l` shows the Ally's audio output device
- [ ] NVMe is visible as a block device
- [ ] Battery and thermal show up in `/sys/class/`
- [ ] `playos_input.h` is defined and compiles cleanly
- [ ] Prototype evdev backend reads controller input on the Ally
- [ ] `make ally-usb-image` produces a bootable USB image
- [ ] ROG Ally kernel config committed and reviewed

---

## Repositories Primarily Involved

| Repo | Work |
|---|---|
| `playos-refdistro` | ROG Ally kernel config, firmware integration, USB image target, device tests |
| `playos-platform-api` | `playos_input.h` C ABI definition, evdev backend prototype |
| `playos-spec` | ROG Ally hardware profile, input mapping document |

---

## Testing Approach

- Physical ROG Ally required for all tests in this sprint
- Device enumeration scripts run at boot and write results to `/run/playos/hw-check.log`
- Input test: run `evtest` on the controller node; verify all logical buttons
- Audio test: play a 1-second sine tone through `aplay`
- QEMU path: unchanged from Sprint 2, continues to work in CI

---

## Exit Gate

A PlayOS USB image boots on physical ROG Ally hardware. All essential devices enumerate correctly. The `libplayos` input C ABI is defined, prototype backend compiles, and a test program reads button presses from the Ally controller.

*Previous: [Sprint 2](Sprint-2.md) | Next: [Sprint 4](Sprint-4.md)*
