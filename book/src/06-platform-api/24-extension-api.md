# Extension API

The Extension API provides a mechanism for vendor extensions, device-specific
features, and optional platform modules to expose functionality beyond the
standard Platform API surface without modifying the core headers.

## Purpose

- **Vendor extensions** — expose device-specific features (fan control, TDP,
  RGB lighting) without polluting the standard API.
- **Optional modules** — modules that are not part of the core API but are
  provided by a conformant implementation.
- **Future-proofing** — allow experimentation before promoting a feature to
  the standard API.

## Mechanism

The extension mechanism is deliberately simple: a string-keyed function
registry that backends populate at startup.

```cpp
namespace PlayOS {
namespace Extension {

// Query whether an extension function is available.
bool Has(const std::string& extensionId);

// Call an extension function by name. Returns nullptr if not available.
// The caller must know the expected signature.
void* GetFunction(const std::string& extensionId);

} // namespace Extension
} // namespace PlayOS
```

## Extension naming

Extension identifiers follow the same namespaced convention as capabilities:

```
vendor.<vendor-id>.<feature>
```

Examples:
- `vendor.rog-ally.fan-control` — ROG Ally fan curve control
- `vendor.rog-ally.tdp-control` — ROG Ally TDP management
- `vendor.steam-deck.back-buttons` — Steam Deck grip button mapping

## Usage pattern

```cpp
if (PlayOS::Extension::Has("vendor.rog-ally.tdp-control")) {
    auto setTDP = reinterpret_cast<void(*)(int)>(
        PlayOS::Extension::GetFunction("vendor.rog-ally.tdp-control")
    );
    if (setTDP) {
        setTDP(15); // 15W TDP
    }
}
```

## When to use extensions vs. capabilities

- **Capability** — a standard feature that the platform defines and that
  applications query before using. Capabilities have a stable API in the
  Platform API headers.
- **Extension** — a vendor-specific feature with no standard API. Extensions
  are discovered at runtime and called through the function registry.

An extension MAY be promoted to a capability with a standard API via the
RFC process.

## Requirements

- Extensions MUST be optional — no application should require one to
  function.
- Extensions MUST NOT duplicate standard API functionality.
- `GetFunction` MUST return `nullptr` for unknown or unavailable extensions.
- Extension function signatures are convention-based; the platform does not
  validate them at compile time.

## Related

- [Vendor Capabilities](../05-capability-model/09-vendor-capabilities.md)
- [API Design Rules](02-api-design-rules.md)
- [Backend Porting Model](../12-device-model-and-porting/03-backend-porting-model.md)
