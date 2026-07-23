# Marketplace API

The Marketplace API provides store browsing, purchase, and entitlement
checking from within an application. It is a **runtime-only** feature gated
on the optional capability `platform.marketplace` (enum:
`Capability::Marketplace`).

## Purpose

- **In-app purchases** — buy DLC, expansions, or virtual currency.
- **Entitlement checking** — verify the player owns a product.
- **Store navigation** — open the store to a specific product or category.
  This typically activates the overlay rather than rendering in-app.

## API concepts

```cpp
namespace PlayOS {
namespace Marketplace {

// Check whether the current player owns a product.
bool HasEntitlement(const std::string& productId);

// Open the store to a specific product (activates the shell overlay).
void ShowProduct(const std::string& productId);

// Open the store to a category or search.
void ShowStore(const std::string& category = "");

} // namespace Marketplace
} // namespace PlayOS
```

## Store UX

The marketplace is displayed by the **shell**, not the application. When
`ShowProduct` or `ShowStore` is called, the shell activates its overlay
surface and renders the store UI. The application is paused (see
[Overlay API](15-overlay-api.md)). This ensures a consistent store
experience across all titles.

## Entitlements

Entitlements are granted when a player purchases or redeems a product.
`HasEntitlement` checks the local entitlement cache, which is synced from
the cloud on startup and when the store overlay closes.

DLC and add-on content MAY be delivered as separate `.gpk` packages or
as content within the base package, unlocked by entitlement.

## Capability

| Capability | Enum | Status |
|---|---|---|
| `platform.marketplace` | `Capability::Marketplace` | Optional (runtime-only) |

## Failure behavior

When `Capability::Marketplace` is absent:
- `HasEntitlement()` returns `false`.
- `ShowProduct()` and `ShowStore()` are no-ops.

## Requirements

- Entitlements MUST be verified server-side before granting in-app content —
  `HasEntitlement` is a convenience, not a security boundary.
- Store navigation MUST pause the application and activate the overlay.
- The marketplace MUST support multiple store sources (official, community,
  private — see [Store Sources](../11-cloud-and-marketplace/08-store-sources.md)).

## Related

- [Marketplace Catalog](../11-cloud-and-marketplace/07-marketplace-catalog.md)
- [Entitlements](../11-cloud-and-marketplace/09-entitlements.md)
- [Overlay API](15-overlay-api.md)
- [Shell and UX](../09-shell-and-ux/01-overview.md)
- RFC-0004 (Platform API Surface)
- RFC-0007 (Cloud and Marketplace Architecture)
