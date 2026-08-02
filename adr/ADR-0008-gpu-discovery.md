# ADR-0008 — PCI Enumeration for GPU Selection

**Date:** Sprint 4  
**Status:** Accepted  
**Deciders:** PlayOS core team

---

## Context

The compositor needs to select the correct DRM device. The simplest approach is to hardcode `/dev/dri/card0`. The correct approach is enumeration.

## Decision

Always enumerate DRM devices and select by PCI vendor identity and active display connector. Never hardcode `/dev/dri/card0` or any other device path.

## Rationale

- **`card0` is not guaranteed:** On systems with multiple DRM devices, or depending on driver load order, the integrated GPU may not be `card0`
- **Intel expansion (Sprint 13):** Supporting Intel graphics requires that the compositor works without knowing the GPU vendor at compile time — enumeration is the only correct approach
- **Future-proofing:** If PlayOS ever supports dual-GPU setups (dGPU + iGPU), enumeration is required
- **Correctness on real hardware:** Tested on the ROG Ally, where `card0` is currently correct, but this should not be relied upon

## Selection Algorithm

```
For each DRM device in drmGetDevices2():
    Resolve PCI vendor ID
    Check if a connector on this device is connected to an active display
    If connected:
        Select this device
        Break
If no connected device found:
    Select first AMD (0x1002) device
    Else select first Intel (0x8086) device
    Else select first valid DRM device
    Else fatal error
```

Log the selected device path, PCI ID, connector name, and preferred mode.

## Consequences

- Compositor startup takes slightly longer (one DRM enumeration call)
- Code is more robust on all hardware configurations
- The selection algorithm must be tested on: single AMD GPU, single Intel GPU, and eventually multi-GPU setups
- `PCI_VENDOR_AMD = 0x1002` and `PCI_VENDOR_INTEL = 0x8086` are defined in compositor code
