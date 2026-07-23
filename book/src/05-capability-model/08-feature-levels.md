# Feature Levels

Not every capability is binary. Some features exist on a spectrum: a device
may support single-touch but not multi-touch, or stereo audio but not
spatial audio. **Feature levels** express this gradation within a single
capability.

## Levels vs separate capabilities

A feature level refines a capability rather than replacing it. `input.touch`
with level `single` means "touch is present, but only one finger." A
separate capability `input.multi-touch` would fragment the namespace and
require applications to check both. Feature levels keep the namespace clean.

## How levels work

```cpp
// Is touch present at all?
bool hasTouch = PlayOS::Capabilities::Has(PlayOS::Capability::Touch);

// What level of touch is available?
auto level = PlayOS::Capabilities::Level(PlayOS::Capability::Touch);
// level might be CapabilityLevel::SingleTouch or CapabilityLevel::MultiTouch
```

- `Has(capability)` returns `true` if the capability is present at any
  level.
- `Level(capability)` returns the specific level, or `Level::None` if the
  capability is absent.
- Levels are ordered: a higher level implies all lower levels. A device
  with `MultiTouch` can satisfy `SingleTouch`.

## When to use feature levels

Feature levels are appropriate when:

1. The capability is naturally graded (touch, audio quality, display
   features).
2. Applications would benefit from graceful degradation rather than
   all-or-nothing gating.
3. The levels represent increasing capability, not unrelated features.

## When NOT to use feature levels

- If the levels are unrelated features, use separate capabilities.
- If there are only two meaningful states, use a boolean capability.
- If levels would complicate the API without practical benefit, keep it
  simple.

## Example levels

| Capability | Levels |
|---|---|
| `input.touch` | `single`, `multi-2`, `multi-5`, `multi-10` |
| `display.hdr` | `hdr10`, `dolby-vision` |
| `audio.spatial` | `stereo`, `surround-5.1`, `surround-7.1`, `atmos` |

## Related

- [What Is a Capability?](02-what-is-a-capability.md)
- RFC-0003 (Capability Model — unresolved question on feature levels)
