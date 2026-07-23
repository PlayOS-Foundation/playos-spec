# Marketplace

The **PlayOS Marketplace** is the application store. It is the primary
way players discover, purchase, and install applications.

## What the marketplace provides

| Feature | Description |
|---|---|
| Catalog | Searchable, filterable list of applications |
| Product pages | Screenshots, trailers, description, ratings |
| Purchase | Credit card, PayPal, wallet, gift codes |
| Downloads | .gpk download with resume support |
| Updates | Automatic or manual application updates |
| Reviews | Player ratings and reviews |
| Refunds | 14-day / 2-hour refund window |
| Wishlist | Save applications for later |
| Gifting | Purchase applications for friends |

## Marketplace components

```
┌────────────────────────────────────────────┐
│            Marketplace UI (Shell)          │  ← On-device UI
├────────────────────────────────────────────┤
│           Marketplace Catalog API          │  ← REST API
├────────────────────────────────────────────┤
│        Marketplace Service (Cloud)         │  ← Catalog, search, reviews
├────────────────────────────────────────────┤
│          Package Storage (Cloud)           │  ← .gpk files
├────────────────────────────────────────────┤
│          CDN (Content Delivery)            │  ← Fast downloads
└────────────────────────────────────────────┘
```

## Revenue model

The default revenue split is 85% developer / 15% PlayOS Foundation.
This applies only to the official PlayOS Marketplace. Self-hosted and
OEM stores set their own splits.

## Curation

The marketplace is primarily **algorithmically curated**:

- New releases
- Popular (by downloads and play time)
- Personalized recommendations
- Editorial features (staff picks)

There is no manual approval gate for publishing. Applications are
published immediately after automated validation.

## Multiple stores

The marketplace is one store source among potentially many. Players
can add OEM stores, self-hosted stores, and private stores. See
[Store Sources](../11-cloud-and-marketplace/08-store-sources.md).

## Related

- [Marketplace Catalog](../11-cloud-and-marketplace/07-marketplace-catalog.md)
- [Payments and Payouts](../11-cloud-and-marketplace/10-payments-and-payouts.md)
- [Entitlements](../11-cloud-and-marketplace/09-entitlements.md)
- [Store Sources](../11-cloud-and-marketplace/08-store-sources.md)
- [OEM Stores](../11-cloud-and-marketplace/18-oem-stores.md)
