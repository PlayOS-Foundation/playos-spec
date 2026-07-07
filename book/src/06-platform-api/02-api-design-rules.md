# API Design Rules

Every addition to the Platform API must follow a small set of rules that keep
the surface stable, portable, and engine-agnostic.

## 1. No engine dependency

Public Platform API headers MUST NOT `#include` a game-engine header. No
Raylib, SDL, Godot, bgfx, or custom-engine types or macros may appear in the
public surface. Engine integration is the responsibility of the
[Engine Integration](../07-engine-integration/01-overview.md) layer, not the
Platform API.

## 2. Capabilities, not platform checks

Feature availability MUST be discovered at runtime through the capability
model (see [Capabilities API](08-capabilities-api.md)). Do not branch on the
operating system, CPU architecture, or device name in public API or
application code. The Platform API implementation hides those details behind
[backends](../12-device-model-and-porting/03-backend-porting-model.md).

## 3. Stable namespaces and names

All public symbols live in `namespace PlayOS`. Modules use nested namespaces
(`Input`, `Capabilities`, `Lifecycle`). Enumerations use `enum class` for
type safety. Names must be stable; renaming a public symbol is a breaking
change.

## 4. Always define failure behavior

Every API must document what happens when it cannot succeed. The platform
must fail predictably (a well-defined error, a documented no-op, or a
documented fallback), never crash or leave undefined state.

## 5. Additive changes preferred

Adding a new module, capability, button, or backend is preferred over
modifying an existing one. Breaking changes must follow the
[API Versioning](25-api-versioning.md) and
[Deprecation Policy](../16-governance-and-process/08-deprecation-policy.md).

## 6. One public header per module

Each module lives in one header under `include/playos/`. Implementation
details (including backend interfaces) stay in `src/` and must not leak into
the public headers.

## 7. Keep it small

The Platform API is a set of minimal platform services, not a general-purpose
OS abstraction. If a feature belongs in an engine, it does not belong here.
