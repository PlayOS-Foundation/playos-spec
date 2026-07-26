---
rfc: 0020
title: Typography and Font System
status: Proposed
authors: []
created: 2026-07-26
---

# RFC-0020: Typography and Font System

> **Status:** Proposed
> This RFC proposes a new Platform API module for font discovery, loading,
> and text metrics.

## Summary

Add a `PlayOS::Font` module to the Platform API that lets applications
discover system fonts, load custom fonts from packages, query glyph
metrics (advance, line height, kerning), and rasterize text. Font
support is gated by `font.system` (required) and `font.custom`
(optional) capabilities.

## Motivation

Every PlayOS application renders text — menus, UI, subtitles,
leaderboards, achievement notifications. Without a Platform API for
fonts, each game must bundle its own font rasterizer (FreeType, stb_truetype)
or rely on engine-specific APIs, breaking the engine-agnostic contract.

The Platform API must provide:
- **System font discovery** so the Shell and games can use the same
  typeface without duplication.
- **Custom font loading** so games can ship branded fonts in their
  .gpk packages.
- **Text measurement** so games can lay out UI correctly across
  different display sizes and safe areas.
- **Capability gating** so constrained devices (embedded, low-RAM)
  can report limited font support.

## Guide-Level Explanation

### Why a Font module?

PlayOS applications currently handle text through their engine. A game
using Raylib calls `DrawText()`, a game using SDL calls `TTF_RenderText()`.
Neither path goes through the Platform API, so the Shell cannot
guarantee font consistency, accessibility scaling, or proper safe-area
layout.

The `Font` module provides a single abstraction:

```cpp
// Discover available system fonts.
auto fonts = PlayOS::Font::ListSystemFonts();
// fonts is a vector<FontDescriptor> with family, style, weight.

// Load a custom font bundled in the game's .gpk.
auto myFont = PlayOS::Font::Load("assets/fonts/MyFont.ttf");

// Measure text for layout.
auto metrics = myFont->MeasureText("Hello, PlayOS!", 24.0f);
// metrics.width, metrics.height, metrics.ascender, metrics.descender

// Get the system default font.
auto defaultFont = PlayOS::Font::SystemDefault();
```

### Capabilities

| Capability | Status | Description |
|---|---|---|
| `font.system` | Required | System font discovery and default font access. |
| `font.custom` | Optional | Loading custom fonts from package assets. |

All Runtime Devices MUST support `font.system`. SDK Targets SHOULD
support `font.system`; if they cannot (e.g., a headless test
environment), they report it absent. `font.custom` is optional —
devices with limited storage may restrict custom font loading.

### Font prioritization

The Font module uses a defined priority for resolving font families:

1. **Custom fonts** loaded by the application (highest).
2. **System fonts** provided by the Runtime.
3. **Fallback font** — the Runtime's built-in fallback (always present,
   always monospace).

## Reference-Level Explanation

### Namespace and types

```cpp
namespace PlayOS {

enum class FontWeight { Thin, Light, Regular, Medium, Bold, Black };
enum class FontStyle { Normal, Italic, Oblique };

struct FontDescriptor {
    std::string family;        // e.g., "Inter", "Noto Sans"
    FontWeight weight;         // Regular
    FontStyle style;           // Normal
    bool monospace;            // true if monospaced
};

struct TextMetrics {
    float width;               // total advance width
    float height;              // line height (ascent + descent + leading)
    float ascender;            // distance from baseline to top
    float descender;           // distance from baseline to bottom (positive)
    float lineGap;             // recommended gap between lines
};

class FontFace {
public:
    TextMetrics MeasureText(std::string_view text, float size) const;
    FontDescriptor GetDescriptor() const;
};

namespace Font {
    // Required (font.system capability)
    std::vector<FontDescriptor> ListSystemFonts();
    std::unique_ptr<FontFace> SystemDefault();
    std::unique_ptr<FontFace> FromDescriptor(const FontDescriptor& desc);

    // Optional (font.custom capability)
    std::unique_ptr<FontFace> Load(const std::string& path);
}

} // namespace PlayOS
```

### Loading model

- `SystemDefault()` returns the Runtime's default UI font. This is the
  typeface used by the Shell for menus, notifications, and settings.
  Applications SHOULD use this for UI consistency.
- `FromDescriptor()` returns the best-matching system font for the
  requested family/weight/style. If no exact match, returns the
  closest available font.
- `Load()` loads a font file from the application's package assets.
  Path is relative to the package root. Supported formats: TrueType
  (.ttf), OpenType (.otf). WOFF2 support is OPTIONAL.

### Text measurement

`MeasureText()` returns metrics for the given string at the given
point size. Metrics are independent of the rendering API — they
describe the font geometry, not rasterization.

- `width` is the total horizontal advance.
- `height` is the line height (ascent + descent + line gap).
- `ascender` / `descender` are relative to the baseline.

### Fallback behavior

If `font.system` is reported absent:
- `ListSystemFonts()` returns an empty vector.
- `SystemDefault()` returns the fallback monospace font.
- `FromDescriptor()` returns the fallback font.

This ensures applications always have at least one usable font.

### Internationalization

System fonts MUST support Latin, Cyrillic, and CJK character sets to
at least basic coverage. Applications requiring broader Unicode
coverage SHOULD bundle custom fonts via `Load()`.

## Drawbacks

- **Adds surface area to the Platform API.** Each new module requires
  maintenance, conformance tests, and backend implementations.
- **Font licensing complexity.** System fonts must be freely
  redistributable. The reference Runtime should use open-licensed
  fonts (e.g., Inter, Noto).
- **Rasterization is not included.** This API provides metrics only;
  actual glyph rasterization remains engine-specific. A future RFC
  could add `Font::RasterizeGlyph()`.

## Rationale and Alternatives

### Why not leave fonts to engines?

Leaving fonts to engines means:
- No guarantee of consistent typography across the Shell and
  applications.
- No font discovery — games hardcode font paths or bundle fonts
  regardless of system availability.
- No accessibility scaling — each engine implements text sizing
  differently.

### Why not include rasterization?

Rasterization is engine-specific (GPU texture atlas, CPU bitmap, SDF).
Forcing one rasterization method into the Platform API would violate
engine-agnosticism. Future RFCs may add optional rasterization
backends.

### Why capabilities?

Not all devices need custom font loading. Low-RAM devices (embedded,
retro handhelds) may only support the system fallback font. The
capability model lets the specification define the full API while
allowing constrained devices to report reduced support.

## Prior Art

- **SDL_ttf** — loads TTF fonts and renders to surfaces. Couples
  loading and rasterization.
- **stb_truetype** — single-header TTF parser with metrics. No
  discovery.
- **DirectWrite** (Windows) — font discovery, metrics, and
  rasterization. Tightly integrated with DWrite/D2D.
- **CoreText** (macOS/iOS) — font discovery, metrics, layout. Apple-
  specific.
- **fontconfig** (Linux) — font discovery and matching. No metrics API.
- **Android Canvas/Typeface** — font loading and metrics via Java/Kotlin.

PlayOS Font API takes the "metrics without rasterization" approach
similar to fontconfig + FreeType query, but exposed as a simple C++
API with capability gating.

## Unresolved Questions

- Should the API include ligature and kerning information in
  `MeasureText()`?
- Should `ListSystemFonts()` support filtering by Unicode coverage?
- Should the Shell expose a "user preferred font size" through a
  separate API (accessibility scaling)?
- What is the canonical list of required character sets for system
  fonts?

## Future Possibilities

- **Font rasterization API** — `Font::RasterizeGlyph()` returning
  bitmap or SDF data.
- **Text layout API** — multi-line text with alignment, wrapping,
  and rich formatting.
- **Font download service** — Cloud-hosted font catalog for
  applications to request additional fonts at runtime.
- **Variable font support** — weight/width/optical size axes.
- **Emoji font support** — system emoji fallback chain.
