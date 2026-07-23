# Marketplace Catalog

The **Marketplace Catalog** is the searchable index of all published
PlayOS applications. It powers the store UI on the device and the web
storefront.

## What the catalog contains

| Field | Description |
|---|---|
| App ID | Unique identifier (UUID v7) |
| Title | Display name |
| Developer | Developer ID and display name |
| Description | Short and long descriptions |
| Category | Genre, tags |
| Rating | Age rating (ESRB, PEGI) |
| Screenshots | Up to 10 images |
| Trailer | Video URL |
| Price | Free or numeric |
| Release date | When it was published |
| Version | Latest version number |
| Size | Download size |
| Permissions | Required permissions |
| Supported languages | Locale list |
| Minimum target level | Platform API Compatible or higher |

## Catalog API

The catalog is accessed via a REST API at `GET /catalog/v1/apps`:

```json
{
  "apps": [
    {
      "id": "0193a8b2-...",
      "title": "Example Game",
      "developer": { "id": "0193a8b2-...", "name": "Example Studio" },
      "description": "A game about examples.",
      "category": "action",
      "rating": "E",
      "screenshots": ["https://cdn.playos.org/screenshots/abc.png"],
      "price": 499,
      "version": "1.2.0",
      "size": 524288000,
      "permissions": ["storage.persistent", "network.client"],
      "minTargetLevel": "platform-api-compatible"
    }
  ],
  "total": 1,
  "nextCursor": null
}
```

## Search and filtering

The catalog supports:

- Full-text search (`?q=racing`)
- Category filter (`?category=action`)
- Price filter (`?maxPrice=1000`)
- Sorting (`?sort=newest`, `?sort=popular`, `?sort=price`)
- Pagination (`?cursor=...&limit=20`)

## Front page

The front page is a curated list of featured applications, new
releases, and personalized recommendations. The curation is managed
through the Developer Portal.

## Self-hosting

The Marketplace Catalog is a static index backed by a PostgreSQL
database. Self-hosted instances serve the same API. The catalog can
include applications from the public PlayOS Marketplace and private
applications.

## Related

- [Store Sources](08-store-sources.md)
- [Package Storage](06-package-storage.md)
- [Federation and Private Stores](17-federation-and-private-stores.md)
- [Marketplace API](../06-platform-api/19-marketplace-api.md)
