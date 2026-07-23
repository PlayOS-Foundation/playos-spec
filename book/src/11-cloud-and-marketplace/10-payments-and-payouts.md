# Payments and Payouts

The **Payments** service handles application purchases and developer
payouts. It is the only PlayOS Cloud service that cannot be fully
self-hosted (requires a payment processor).

## Purchase flow

1. Player selects an application in the store.
2. Player confirms the purchase (price, permissions).
3. The Payment service creates a payment intent with the payment
   processor (e.g., Stripe).
4. Player completes payment (card, wallet, etc.).
5. Payment processor confirms the payment.
6. An entitlement is created for the player.
7. The application is downloaded and installed.

## Supported payment methods

| Method | Availability |
|---|---|
| Credit/debit card | Global |
| PayPal | Most regions |
| PlayOS Wallet | Global (prepaid balance) |
| Gift codes | Global |
| Carrier billing | Select carriers |

## Revenue split

The default revenue split is:

| Party | Share |
|---|---|
| Developer | 85% |
| PlayOS Foundation | 15% |

For self-hosted stores, the store operator sets their own split.
The PlayOS Foundation does not take a cut from self-hosted store
transactions.

## Payouts

Developers receive payouts monthly (net 30). Payouts are sent via:

- Bank transfer (ACH, SEPA)
- PayPal
- Stripe Connect

Minimum payout threshold: $50 USD.

## Free applications

Free applications go through the same purchase flow but with a $0
price. An entitlement is still created — this enables "purchasing"
free applications once and downloading them on all linked devices.

## Refunds

Players can request a refund within 14 days of purchase if they have
played less than 2 hours. This matches common platform policies.
Refunds revoke the entitlement.

## Related

- [Entitlements](09-entitlements.md)
- [Marketplace Catalog](07-marketplace-catalog.md)
- [Developer Portal](05-developer-portal.md)
- [Self-Hosting Principles](02-self-hosting-principles.md)
