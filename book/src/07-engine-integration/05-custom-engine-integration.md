# Custom Engine Integration

A developer writing their own engine — or using a proprietary one — follows
the same engine-agnostic contract as Raylib or SDL. This chapter provides
guidelines.

## The minimum integration

To integrate any custom engine with the PlayOS Platform API:

1. **Include both headers.** `engine.h` (or equivalent) and
   `playos/playos.h` in the same source file.
2. **Initialize both.** Engine initialization first, then
   `Lifecycle::Init()`.
3. **Call Update each frame.** `Lifecycle::Update()` once per frame inside
   the engine's main loop.
4. **Shut down both.** `Lifecycle::Shutdown()` before engine shutdown.
5. **Query capabilities.** Use `Capabilities::Has()` before calling
   optional API functions.

## Checklist

| Requirement | What to do |
|---|---|
| Separate headers | Keep engine and Platform API headers isolated |
| Frame loop | Call `Lifecycle::Update()` once per frame |
| Input | Use `Input::Pressed/Down/GetAxis` for console controls |
| Capabilities | Guard optional features with `Capabilities::Has()` |
| Storage | Use `Storage::SavePath()` and `Storage::ConfigPath()` |
| Errors | Follow [Capability Failure Behaviour](../05-capability-model/06-capability-failure-behaviour.md) |
| No engine types in platform calls | Do not pass engine types to Platform API functions |
| No platform types in engine calls | Do not use `Button` or `Axis` in engine rendering code |

## Rendering boundaries

The Platform API does not provide a rendering surface. The custom engine
owns the window and rendering context. The Platform API's `Display::Current()`
provides display properties (size, refresh rate, safe area) that the engine
may use to configure its viewport. See
[Rendering Boundaries](08-rendering-boundaries.md).

## Input bridging

If the custom engine has its own input system (keyboard, mouse, raw
controller), it should feed that input into the Platform API's logical
button system through an input bridge. This ensures `Home` and
`QuickSettings` work consistently. See [Input Bridging](07-input-bridging.md).

## Related

- [Engine-Agnostic Contract](02-engine-agnostic-contract.md)
- [Engine Integration Guidelines](09-engine-integration-guidelines.md)
- [Raylib Reference Kit](03-raylib-reference-kit.md)
