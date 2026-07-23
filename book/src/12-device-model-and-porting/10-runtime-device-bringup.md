# Runtime Device Bringup

This chapter is a step-by-step checklist for bringing PlayOS to a
new device as a full Runtime Device — the shell runs natively and
games execute on-device.

## Prerequisites

Before starting bringup, confirm:

- [ ] The device runs Linux (or can be made to).
- [ ] A compositor can be installed (wlroots-based preferred).
- [ ] Controller input is exposed as evdev devices or can be made to.
- [ ] The device has a GPU with working DRM/KMS or EGL drivers.
- [ ] Audio output is available via PipeWire, PulseAudio, or ALSA.

## Phase 1: Device profile

1. [ ] Write a `device-profile.toml` per [Device Profile Schema](
   02-device-profile-schema.md) and [RFC-0006](
   ../../rfcs/0006-device-profile-format.md).
2. [ ] Map all physical buttons to symbolic names in `[input]`.
3. [ ] Set display dimensions, refresh rate, and safe areas in `[display]`.
4. [ ] Declare all capabilities in `[capabilities]`.
5. [ ] Place the profile in `/etc/playos/device-profiles/<id>.toml`.
6. [ ] Verify the profile is loadable: `playos-cli profile validate`.

## Phase 2: Backend implementations

For each Platform API module, implement the backend interface. Start
with the null backend as a template, then implement platform-specific
code.

| Module | Backend | Required? | How to test |
|---|---|---|---|
| Input | `LinuxInputBackend` | ✅ Required | `playos-cli test input` |
| Display | `LinuxDisplayBackend` | ✅ Required | `playos-cli test display` |
| Storage | `LinuxStorageBackend` | ✅ Required | `playos-cli test storage` |
| Battery | `LinuxBatteryBackend` | Handheld only | Check sysfs |
| Network | `LinuxNetworkBackend` | Recommended | `playos-cli test network` |
| Audio | Linux audio backend | Recommended | Test volume/mute |
| Bluetooth | `LinuxBluetoothBackend` | Optional | Check `/sys/class/bluetooth` |

## Phase 3: Build system integration

1. [ ] Add a CMake platform preset for the device.
2. [ ] Select the correct backend sources for each module.
3. [ ] Build: `cmake --preset <device> && cmake --build build/<device>`.
4. [ ] Verify the binary links without missing symbols.

## Phase 4: Conformance testing

1. [ ] Run `playos-cli test --suite conformance`.
2. [ ] Verify all expected capabilities are registered.
3. [ ] Verify input: every physical button maps to the correct PlayOS Button.
4. [ ] Verify display: dimensions match the physical screen.
5. [ ] Verify battery: level and charging status are correct.
6. [ ] Verify storage: saves and config paths are writable.
7. [ ] Verify network: Wi-Fi scan returns visible networks.

## Phase 5: Integration testing

1. [ ] Boot the PlayOS Shell on the device.
2. [ ] Verify the shell renders at the correct resolution.
3. [ ] Navigate through Home, Library, Store, and Settings screens.
4. [ ] Test suspend/resume (if supported).
5. [ ] Install and launch a game package.
6. [ ] Test gamepad input in-game.
7. [ ] Test quit → back to shell flow.

## Phase 6: Certification readiness

If the device targets certification, verify against
[Certified Devices](../04-target-model/05-certified-devices.md):

- [ ] All required capabilities pass conformance.
- [ ] Frame timing meets the certification threshold.
- [ ] Suspend/resume works within the required latency budget.
- [ ] Security model is intact (process sandbox, storage isolation).
- [ ] Device profile is submitted to the certification repository.

## After bringup

Once the device passes all phases:

1. [ ] Add the device profile to the reference devices repository.
2. [ ] Add a per-device chapter to [this part](01-overview.md).
3. [ ] Report backend coverage gaps to the issue tracker.

## Related

- [SDK Target Bringup](11-sdk-target-bringup.md) — lighter-weight bringup
- [Backend Porting Model](03-backend-porting-model.md)
- [Target Model](../04-target-model/01-overview.md)
- [ROG Ally Reference](12-rog-ally-reference.md) — worked example
