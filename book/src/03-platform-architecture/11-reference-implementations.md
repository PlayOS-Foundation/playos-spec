# Reference Implementations

The PlayOS project maintains **reference implementations** of key
components. These are functional, tested implementations that serve
as the standard against which other implementations are measured.

## What reference implementations are

A reference implementation:

- Implements the specification completely and correctly.
- Is written for clarity, not maximum performance.
- Serves as documentation — if the spec is ambiguous, see how the
  reference does it.
- Is used in the certification test suite.
- Is maintained alongside the specification.

A reference implementation is **not**:

- The only allowed implementation.
- Production-hardened (though it may be suitable for production).
- A requirement — conformant implementations can be completely
  independent.

## Current reference implementations

| Component | Repository | Language |
|---|---|---|
| Platform API headers | `playos-platform-api` | C++ |
| Runtime | `playos-runtime` | C++ |
| Shell | `playos-shell` | C (Raylib) |
| Cloud services | `playos-cloud` | Rust |
| Tools | `playos-tools` | Rust |
| Device profiles | `playos-reference-devices` | TOML |

## Reference engine

Raylib is the **reference rendering engine**. The reference Shell and
many code examples use Raylib. This is a convenience choice — Raylib
is small, fast, and easy to set up. The Platform API does not depend
on Raylib.

## Reference hardware

| Device | Type | Repository |
|---|---|---|
| ROG Ally | Handheld PC | `playos-reference-devices/rog-ally/` |
| Generic Linux PC | Desktop/Laptop | `playos-reference-devices/generic-linux-pc/` |

See [Reference Devices](../04-target-model/04-reference-devices.md).

## Adding a reference implementation

1. Propose via RFC.
2. Implement in a new repository under `PlayOS-Foundation/`.
3. Add to the CI/test matrix.
4. Document in this chapter.
5. Link from the relevant specification sections.

## Related

- [Layered Architecture](02-layered-architecture.md)
- [Platform Specification](03-platform-specification.md)
- [Reference Devices](../04-target-model/04-reference-devices.md)
- [Reference Shell](../09-shell-and-ux/16-reference-shell.md)
