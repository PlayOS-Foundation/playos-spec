# Engine Integration Guidelines

This chapter collects general guidelines for integrating any engine with
the PlayOS Platform API.

## The golden rule

> The engine renders. The Platform API provides platform services. Neither
> includes the other.

Every integration decision follows from this rule.

## Checklist for engine integrators

### Build system

- [ ] Link the Platform API as a dependency (`PlayOS::PlatformAPI`).
- [ ] Do not modify Platform API headers.
- [ ] Provide a CMake snippet or equivalent for downstream projects.

### Lifecycle

- [ ] Call `Lifecycle::Init()` after engine initialization.
- [ ] Call `Lifecycle::Update()` once per frame.
- [ ] Call `Lifecycle::Shutdown()` before engine shutdown.

### Input

- [ ] Map engine input events to PlayOS `Button` and `Axis` values.
- [ ] Ensure `Home` and `QuickSettings` are always mapped.
- [ ] Use backend-level bridging (not application-level translation).

### Display

- [ ] Use `Display::Current()` for physical display properties.
- [ ] Respect safe area insets for critical UI.

### Capabilities

- [ ] Query `Capabilities::Has()` before calling optional API functions.
- [ ] Provide fallback paths when capabilities are absent.

### Storage

- [ ] Use `Storage::SavePath()` and `Storage::ConfigPath()` for
  persistent data.
- [ ] Do not write save data inside the package directory.

### Audio

- [ ] Use `Audio::GetMasterVolume()` and `Audio::SetMasterVolume()` for
  system volume control.
- [ ] Respect mute state.

### Package integration

- [ ] Bundle engine runtime libraries in the `.gpk` package.
- [ ] Declare all required permissions in the manifest.
- [ ] Test on all target architectures.

## Sample structure

Every engine integration kit SHOULD include:

```
playos-engine-<name>/
├── CMakeLists.txt           # Build integration
├── include/
│   └── playos-engine-<name>/
│       └── bridge.h         # Input bridge (if needed)
├── src/
│   └── bridge.cpp
├── samples/
│   └── hello/
│       ├── CMakeLists.txt   # Sample build
│       └── main.cpp         # Dual-include pattern
└── README.md
```

## Requirements

- Integration kits MUST NOT modify Platform API headers.
- Integration kits MUST demonstrate the dual-include pattern.
- Samples MUST compile and run on at least one reference target.

## Related

- [Engine-Agnostic Contract](02-engine-agnostic-contract.md)
- [Raylib Reference Kit](03-raylib-reference-kit.md)
- [Custom Engine Integration](05-custom-engine-integration.md)
