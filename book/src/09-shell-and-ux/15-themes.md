# Themes

The Shell supports **visual themes** — customizable colors, fonts, and
backgrounds. OEMs can ship custom themes; players can choose from
installed themes.

## Theme components

A theme defines:

| Component | Description |
|---|---|
| **Colors** | Primary, secondary, accent, background, text, error |
| **Typography** | Font family, sizes (body, heading, caption) |
| **Spacing** | Grid gaps, margins, padding |
| **Border radius** | Corner rounding for cards, buttons, dialogs |
| **Shadows** | Drop shadow depth and color |
| **Background** | Home screen wallpaper (static image or slideshow) |
| **Sound** | Navigation and notification sound set |

## Theme format

Themes are TOML files:

```toml
[theme]
name = "Dark"
author = "PlayOS Foundation"
version = "1.0.0"

[colors]
primary = "#FF6B35"
secondary = "#004E89"
accent = "#1A936F"
background = "#1A1A2E"
surface = "#16213E"
text = "#EAEAEA"
textSecondary = "#9A9A9A"
error = "#E63946"

[typography]
fontFamily = "Inter"
bodySize = 24
headingSize = 36
captionSize = 16

[background]
type = "color"
color = "#1A1A2E"
# type = "image"
# path = "/usr/share/playos/themes/dark/background.png"
```

## Built-in themes

PlayOS ships with:

- **Dark** — default. Dark backgrounds, warm accent. Optimized for
  OLED displays.
- **Light** — light backgrounds. For well-lit environments.
- **High Contrast** — maximum contrast for accessibility.

## OEM themes

OEMs can ship custom themes in their device image:

- Place theme files in `/usr/share/playos/themes/<theme-name>/`.
- Register the theme in the device profile.
- Set as the default theme for the device.

## Theme switching

Players change themes in Settings → Personalization → Theme. The
change takes effect immediately with a crossfade transition (200 ms).

## Application theming

Applications are encouraged to respect the system theme. The Platform
API provides theme information:

```cpp
auto theme = Shell::GetTheme();
// theme.colorScheme (Light/Dark)
// theme.accentColor
// theme.highContrast (bool)
```

Applications can use this to match their own UI to the system theme,
but are not required to do so.

## Related

- [Accessibility (High Contrast)](14-accessibility.md)
- [Settings](08-settings.md)
- [OEM Stores](../11-cloud-and-marketplace/18-oem-stores.md)
