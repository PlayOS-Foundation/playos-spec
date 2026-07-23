# Package Metadata

Package metadata extends the manifest with store-facing information:
descriptions, authors, icons, screenshots, and categorization.

## Manifest fields

Metadata is declared in `manifest.json`:

```json
{
  "description": "A fast-paced platformer with hand-drawn art.",
  "author": "Indie Studio",
  "website": "https://studio.example.com",
  "genre": ["platformer", "action"],
  "players": "1-4",
  "contentRating": {
    "system": "ESRB",
    "rating": "E"
  }
}
```

| Field | Description |
|---|---|
| `description` | Short description (recommended: under 280 characters) |
| `author` | Developer or studio display name |
| `website` | Developer website URL |
| `genre` | Array of genre tags for store categorization |
| `players` | Number of players (e.g. `"1"`, `"1-4"`, `"1-16"`) |
| `contentRating` | Content rating system and value |

## Store assets

Store-facing assets live in the package root:

| File | Recommended size | Purpose |
|---|---|---|
| `icon.png` | 512×512 | Square icon for library and store listings |
| `banner.png` | 1920×640 | Wide banner for store hero/featured placement |
| `screenshots/` | 1920×1080 | Up to 10 screenshots for the store page |

The marketplace displays these assets on the product page. The runtime
uses `icon.png` in the library and game switcher.

## Content rating

PlayOS supports multiple rating systems. The developer declares the rating
in the manifest; the marketplace may require third-party rating
verification before listing.

```json
"contentRating": {
  "system": "ESRB",
  "rating": "E10+"
}
```

Supported systems: `ESRB`, `PEGI`, `USK`, `IARC`, `CERO`.

## Related

- [Manifest](04-manifest.md)
- [Marketplace Catalog](../11-cloud-and-marketplace/07-marketplace-catalog.md)
