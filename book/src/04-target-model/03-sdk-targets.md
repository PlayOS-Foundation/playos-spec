# SDK Targets

An **SDK Target** supports the PlayOS Platform API, or a subset of it,
**without running the full PlayOS runtime.**

SDK targets let developers build and test PlayOS applications on platforms
that already have their own operating system, and let PlayOS games reach
platforms beyond dedicated runtime devices.

## What an SDK target provides

SDK targets typically support the portable subset of the Platform API:

- input abstraction (buttons, sticks, touch where available),
- storage (save and config paths),
- display information (size, safe area),
- audio helpers,
- application lifecycle basics.

## What an SDK target may not provide

Runtime-only features are generally unavailable on SDK targets:

- the PlayOS compositor and system overlay,
- OS-level suspend/resume,
- launching other games,
- package installation and the marketplace client,
- device-level power, brightness, TDP, and fan control.

Applications must not assume these exist. They discover availability
through the [Capability Model](../05-capability-model/01-overview.md).

## Reference SDK targets

- **Windows** — primary development target.
- **Linux desktop** — development and testing.
- **macOS** — development target.
- **Android** — mobile SDK target.
- **PS Vita (homebrew)** — retro/homebrew target via VitaSDK (for example
  using a Raylib port with a vitaGL backend), shipped as a `.vpk`.

## Why SDK targets matter

They dramatically lower the barrier to adoption. A developer on Windows can
use the same Platform API they would on a runtime device, and — because
the marketplace is SDK-first — even publish without owning PlayOS hardware.
When they later target a full runtime device, they automatically gain the
additional runtime capabilities.

See also: [Runtime Devices](02-runtime-devices.md) and
[Target Compatibility Levels](08-target-compatibility-levels.md).
