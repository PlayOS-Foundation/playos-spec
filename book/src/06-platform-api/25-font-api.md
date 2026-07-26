# Font API

> **Status:** Stub — this chapter is proposed by
> [RFC-0020](../../rfcs/0020-typography-and-font-system.md) and has not
> been written yet.

The `PlayOS::Font` module provides font discovery, loading, and text
measurement. It is gated by `font.system` (required) and
`font.custom` (optional) capabilities.

## Capabilities

| Capability | Status | Required |
|---|---|---|
| `font.system` | Required | Yes |
| `font.custom` | Optional | No |

## Related

- [RFC-0020: Typography and Font System](../../rfcs/0020-typography-and-font-system.md)
- [Display API](11-display-api.md)
- [Themes (typography)](../09-shell-and-ux/15-themes.md)
- [Accessibility](../14-certification/11-accessibility-requirements.md)
