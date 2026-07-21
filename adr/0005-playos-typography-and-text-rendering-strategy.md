# PlayOS Typography and Text Rendering Strategy

- **ADR:** 0005
- **Title:** PlayOS Typography and Text Rendering Strategy
- **Status:** Accepted
- **Date:** 2026-07-22
- **Deciders:** PlayOS Foundation

## Context

PlayOS targets handheld consoles such as the ASUS ROG Ally and Steam Deck, desktop monitors, and future living-room devices. The reference shell must remain clean and readable across small high-density handheld displays, conventional desktop displays, and larger screens viewed from couch distance.

Typography is a platform-level concern for the reference experience because it affects:

- controller-first usability and readability at distance;
- accessibility and user-selectable text scaling;
- localization and Unicode coverage;
- visual consistency across the shell, settings, marketplace, overlays, and developer surfaces;
- memory use, glyph-atlas construction, and rendering performance;
- certification and reference-device validation.

The PlayOS Shell uses raylib as its initial rendering foundation. raylib provides practical font loading, glyph atlas generation, UTF-8 text rendering, measurement, and drawing, but it does not by itself define a complete typography system. PlayOS must therefore own font-family policy, fallback, layout, scaling, truncation, localization, and future complex-script shaping above the rendering backend.

## Decision

The PlayOS reference shell and official PlayOS system applications will use the following initial font stack:

| Purpose | Font family |
|---|---|
| Primary shell and UI text | **Inter** |
| Terminal, logs, diagnostics, and developer surfaces | **JetBrains Mono** |
| International glyph fallback | **Noto Sans** |

The selected font assets must be distributed under licenses compatible with PlayOS redistribution. Exact font versions and included weights must be pinned by the implementing repository and updated deliberately.

This decision defines the typography of the PlayOS reference experience. Third-party applications are not required to use these fonts unless a future application-design or certification specification says otherwise.

## Rendering and Ownership Model

The initial reference rendering path is:

```text
PlayOS Shell
    -> PlayOS UI and layout layer
    -> PlayOS text subsystem
    -> raylib font atlas and drawing
    -> Wayland client surface
    -> PlayOS compositor
```

raylib is the initial glyph rasterization and drawing backend. PlayOS owns the higher-level text behavior, including:

- semantic text styles and logical font sizes;
- font-family selection and ordered fallback;
- wrapping, clipping, ellipsis, alignment, and measurement;
- device-profile and accessibility scaling;
- localization-aware layout;
- glyph-range and atlas-lifetime policy;
- caching and performance limits;
- future shaping and bidirectional-text integration.

The text subsystem must not expose raylib-specific types as the permanent public PlayOS Platform API. This keeps the rendering implementation replaceable and allows the reference shell to adopt a different renderer or text engine later without redefining application-facing platform contracts.

## Device Profiles and Scaling

Typography must use logical sizes rather than assuming that framebuffer pixels correspond directly to physical readability.

The reference shell must distinguish at least these presentation classes:

| Presentation class | Primary objective |
|---|---|
| Handheld | Readability at arm's length on approximately 7- to 8-inch displays |
| Desktop | Comfortable use on monitors at desk distance |
| TV / couch | Readability at greater viewing distance on larger displays |
| Accessibility large text | User-selected enlargement without loss of navigation or content access |

Resolution alone is not sufficient to select text size. Device profiles, physical display information where available, user preference, and presentation mode should contribute to the effective UI scale.

Reference layouts must be tested at minimum on:

- ASUS ROG Ally-class 1080p handheld displays;
- Steam Deck-class 800p handheld displays;
- 1080p and 1440p desktop monitors, including approximately 32-inch displays;
- a TV/couch profile at common living-room distances.

Text enlargement must preserve focus visibility, navigation order, and access to primary actions. Fixed-height controls must not silently clip enlarged text.

## Font Loading and Atlas Policy

The reference implementation should load a small, intentional set of font faces and weights rather than creating an independent font resource for every rendered size.

Implementations should:

- load fonts at suitable base sizes and scale them only within validated ranges;
- avoid aggressively scaling a low-resolution atlas;
- build glyph ranges from supported locales and required symbols;
- avoid loading all Unicode glyphs into a single atlas by default;
- cache atlases predictably and release unused locale-specific resources;
- disable programming ligatures in terminal and diagnostic surfaces unless explicitly requested;
- preserve clear distinctions between ambiguous monospace glyphs such as `0` and `O`, or `1`, `I`, and `l`.

Signed Distance Field or Multi-channel Signed Distance Field rendering may be introduced for large, animated, or broadly scalable text. It is not required for the first reference-shell implementation.

## Internationalization and Text Shaping

All PlayOS text interfaces must use UTF-8.

Noto Sans is the initial fallback family for scripts not covered by the primary Inter assets. Fallback must be selected per missing glyph or shaped run rather than replacing an entire interface indiscriminately.

Basic font loading and Unicode codepoint rendering do not provide full language support. The text subsystem must be designed so that HarfBuzz or an equivalent shaping engine can be integrated for complex scripts. Bidirectional layout support, such as through FriBidi or an equivalent implementation, may also be required.

A locale must not be declared fully supported until its shaping, bidirectional behavior, line breaking, fallback, truncation, and input methods have been validated. Shipping Noto glyphs alone does not establish complete language support.

## Accessibility

The typography system must support:

- user-selectable text scaling;
- high-contrast themes without loss of text clarity;
- minimum readable sizes defined by presentation profile;
- focus and selection states that do not rely on typography weight alone;
- layouts that reflow or scroll rather than clip essential text;
- readable system messages, dialogs, and recovery screens.

Atkinson Hyperlegible or another accessibility-oriented family may be evaluated as an optional user-selectable profile in a future proposal. It is not the default family established by this decision.

## Visual Identity

Inter provides the initial neutral, modern, and highly readable PlayOS reference identity. JetBrains Mono gives technical surfaces a clear, distinct voice without reducing character legibility. Noto Sans provides a practical foundation for broad script coverage.

A future custom PlayOS typeface may be commissioned or derived from a suitably licensed font after the visual language and glyph requirements have matured. A custom typeface must not reduce accessibility, international coverage, or rendering quality merely to create brand differentiation.

## Consequences

### Positive

- The shell gains a consistent and recognizable typography baseline.
- Inter is suitable for dense handheld UI and larger couch-readable text.
- JetBrains Mono improves clarity in terminals, logs, and diagnostics.
- Noto Sans provides a scalable fallback strategy for internationalization.
- Open redistribution-friendly font choices support self-hosted and downstream PlayOS implementations.
- Typography policy is separated from the initial raylib rendering backend.
- Device-profile scaling becomes an explicit platform responsibility.

### Negative

- Reference images and applications must bundle and version font assets.
- Multiple fallback families and locale glyph sets increase storage and atlas-management complexity.
- Full international support requires shaping and bidirectional-layout work beyond raylib's basic text APIs.
- Text scaling and reflow must be designed into every reference-shell screen rather than added late.
- Visual regression testing is required across device classes, scales, and locales.

### Risks and Mitigations

- **Blurry or uneven text caused by atlas scaling:** validate base atlas sizes and introduce SDF/MSDF only where it provides measurable benefit.
- **Excessive atlas memory use:** use locale-aware glyph subsets, caching, and explicit limits.
- **Missing glyphs:** implement ordered fallback and automated glyph-coverage tests.
- **Incorrect complex-script output:** integrate a shaping engine before claiming support for affected locales.
- **UI breakage at large text sizes:** require reflow, scrolling, and accessibility-profile tests.
- **Font changes altering layout:** pin versions and treat font updates as visual compatibility changes requiring review.

## Alternatives Considered

- **IBM Plex Sans:** strong readability and a technical character, but Inter was selected as the more neutral primary shell family.
- **Atkinson Hyperlegible:** excellent accessibility characteristics, but not selected as the default visual identity. It remains a candidate optional profile.
- **Roboto:** broad ecosystem support, but Inter better matches the intended PlayOS visual direction.
- **Noto Sans as the only UI family:** excellent language coverage, but a primary-plus-fallback model gives the reference shell a more focused visual identity.
- **Cascadia Mono, Fira Code, or Iosevka:** all are credible monospace alternatives. JetBrains Mono was selected for its clear character differentiation and balanced density. Programming ligatures are unnecessary for system diagnostics.
- **A custom PlayOS typeface immediately:** deferred because the platform's visual language, language coverage, and production glyph requirements are not mature enough to justify the maintenance burden.
- **Direct raylib text calls throughout the shell:** rejected because they would couple layout, fallback, scaling, and localization policy to a rendering library and make future evolution harder.

## Non-goals

This decision does not:

- require third-party PlayOS applications to use the reference font stack;
- make raylib part of the permanent public Platform API;
- claim full support for every script covered by Noto Sans;
- define final pixel sizes for every component or device;
- require SDF or MSDF rendering in the first implementation;
- prevent OEM or downstream shells from using a different visual identity while conforming to future accessibility and certification requirements.
