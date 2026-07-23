# Entitlements

**Entitlements** track which applications a player owns. When a player
purchases an application, an entitlement is created linking the
player's account to the application.

## How entitlements work

1. Player purchases an application (free or paid).
2. The Marketplace creates an entitlement record:
   `(playerId, appId, timestamp, source)`.
3. The Runtime checks entitlements before launching an application.
4. The application does not need to check entitlements — the Runtime
   handles it.

## Entitlement sources

| Source | Description |
|---|---|
| `purchase` | Bought through the marketplace |
| `redeem` | Redeemed via code or gift |
| `developer` | Developer access (owns the app) |
| `subscription` | Active PlayOS Plus subscription |
| `preload` | Pre-installed on the device |
| `transfer` | Transferred from another account |

## Offline entitlements

Entitlements are cached locally on the device. A device can validate
entitlements without an internet connection for up to 30 days. After
30 days, the device must reconnect to refresh the entitlement cache.

## Entitlement revocation

Entitlements can be revoked:

- Refund — the player gets their money back, entitlement is removed.
- Fraud — the purchase was fraudulent.
- Developer action — the developer removes the application from sale.

When an entitlement is revoked, the application is no longer
launchable. The player's save data is preserved (in case the
revocation is reversed or they re-purchase).

## Family sharing

PlayOS supports family sharing: up to 5 family members can access a
shared library. The account holder controls who is in the family
group. Applications opted out of family sharing by the developer are
not shared.

## DRM philosophy

PlayOS does not use invasive DRM. Entitlements are a simple ownership
check, not a content protection system. The trust model is:

1. Signed packages prevent tampering.
2. Entitlements prevent casual piracy.
3. Developers who want stronger DRM can implement it themselves.

## Related

- [Accounts and Identity](04-accounts-and-identity.md)
- [Payments and Payouts](10-payments-and-payouts.md)
- [Marketplace API](../06-platform-api/19-marketplace-api.md)
