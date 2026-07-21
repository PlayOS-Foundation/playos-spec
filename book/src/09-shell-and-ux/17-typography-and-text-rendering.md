# Typography and Text Rendering

Typography is a core part of the PlayOS reference experience. Text must remain readable on handheld displays, desktop monitors, and televisions viewed from couch distance, while also supporting accessibility scaling and internationalization.

The normative architecture decision is [ADR-0005: PlayOS Typography and Text Rendering Strategy](../../../adr/0005-playos-typography-and-text-rendering-strategy.md).

## Reference font stack

The PlayOS reference shell and official system applications use:

| Purpose | Reference family |
|---|---|
| Primary shell and UI text | **Inter** |
| Terminal, logs, diagnostics, and developer surfaces | **JetBrains Mono** |
| International glyph fallback | **Noto Sans** |

These fonts define the reference visual identity. Third-party applications are not required to use them unless a future application-design or certification specification says otherwise.

Font versions and included weights must be pinned by the implementing repository. Font updates are visual compatibility changes and require layout and screenshot regression testing.

## Text rendering architecture

The reference shell uses a layered text path:

```text
PlayOS Shell
    -> PlayOS UI and layout layer
    -> PlayOS text subsystem
    -> raylib font atlas and drawing
    -> Wayland client surface
    -> PlayOS compositor
```

raylib is the initial rasterization and drawing backend. It is not the permanent public text API.

The PlayOS text subsystem owns:

- semantic text styles and logical font sizes;
- font selection and ordered fallback;
- measurement, wrapping, clipping, ellipsis, and alignment;
- accessibility and device-profile scaling;
- glyph-range and atlas-lifetime policy;
- localization-aware layout;
- integration with future shaping and bidirectional-text libraries.

Shell code should not scatter direct raylib text calls throughout individual screens. Screens should consume semantic styles such as title, body, label, caption, dialog, and developer-monospace through the PlayOS UI layer.

## Presentation profiles

Text size is determined in logical units. Framebuffer resolution alone does not determine readability.

The reference shell must account for:

- device profile;
- physical display information where available;
- expected viewing distance;
- handheld, desktop, docked, or TV presentation mode;
- user-selected accessibility scale.

The initial validation matrix includes:

| Profile | Validation target |
|---|---|
| Handheld 800p | Steam Deck-class display |
| Handheld 1080p | ASUS ROG Ally-class display |
| Desktop | 1080p and 1440p monitors, including approximately 32-inch displays |
| TV / couch | Common living-room distance and 1080p/4K output |
| Accessibility large text | Enlarged text without loss of navigation or essential actions |

Layouts must reflow, scroll, or otherwise adapt when text is enlarged. Essential labels and actions must not be silently clipped by fixed-height controls.

## Font loading and glyph atlases

The shell should load a small and intentional set of faces and weights. It should not allocate a separate font resource for every visual size.

Implementations should:

- choose base atlas sizes appropriate to validated scale ranges;
- avoid excessive enlargement of low-resolution glyph atlases;
- construct glyph ranges from the active locale and required system symbols;
- avoid loading all Unicode code points into one atlas by default;
- cache locale-specific atlases predictably;
- unload resources that are no longer required;
- disable programming ligatures in diagnostic and terminal views unless explicitly requested.

SDF or MSDF rendering may later be used for large, animated, or broadly scalable text, but it is not required for the first reference implementation.

## Internationalization

All PlayOS text interfaces use UTF-8.

Noto Sans supplies fallback coverage where the selected Inter assets do not contain a required glyph. Fallback should occur per missing glyph or shaped run rather than replacing the entire interface family.

Unicode glyph coverage is not the same as complete language support. Complex scripts may require a shaping engine such as HarfBuzz, while right-to-left and mixed-direction text may require a bidirectional-layout implementation such as FriBidi.

A locale must not be declared fully supported until the following have been validated:

- glyph coverage and fallback;
- shaping and character joining;
- bidirectional layout;
- line breaking and wrapping;
- truncation and ellipsis;
- input methods;
- text scaling and screen reflow.

## Accessibility

The typography system supports user-selectable text scaling and high-contrast presentation. Focus and selection must not rely on font weight alone.

System dialogs, recovery screens, account flows, marketplace screens, and settings must remain readable under the large-text profile. Where content no longer fits, the interface must preserve navigation order and provide scrolling or reflow.

An accessibility-oriented optional font profile, such as Atkinson Hyperlegible, may be proposed later. It is not the default established by ADR-0005.

## Responsibilities and boundaries

The shell and its UI/text subsystem own typography, text layout, and font rendering. The compositor owns output presentation, surface ordering, scaling policy at the Wayland/output boundary, and focus. It does not shape or rasterize shell text.

The portable PlayOS Platform API must not expose raylib font objects, glyph atlases, or wlroots implementation details. This preserves the ability to replace the renderer or text engine without changing application-facing platform contracts.

## Conformance guidance

A reference-shell typography change should be reviewed against:

- handheld and TV readability;
- large-text accessibility behavior;
- supported-locale glyph coverage;
- atlas memory and startup cost;
- screenshot regressions across presentation profiles;
- truncation and wrapping behavior;
- stable developer-console character alignment.

The detailed decision, consequences, alternatives, and non-goals are maintained in [ADR-0005](../../../adr/0005-playos-typography-and-text-rendering-strategy.md).
