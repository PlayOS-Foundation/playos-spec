# Developer Portal

The **Developer Portal** is the web application where developers
manage their PlayOS applications, view analytics, and configure
monetization.

## Features

| Feature | Description |
|---|---|
| Application management | Create, edit, and publish applications |
| Build upload | Upload .gpk files and manage versions |
| Release channels | Manage stable, beta, nightly channels |
| Analytics dashboard | View downloads, revenue, ratings |
| Achievement configuration | Define achievements and icons |
| Leaderboard configuration | Create and manage leaderboards |
| Monetization | Set prices, manage in-app purchases |
| Payout settings | Configure bank/payment details |
| Team management | Invite collaborators, manage roles |
| Support dashboard | Respond to player reviews and reports |

## Application lifecycle

1. **Draft** — created in the portal, not yet submitted.
2. **Review** — submitted for review. PlayOS checks the manifest,
   validates the package, scans for policy violations.
3. **Published** — visible in the catalog, downloadable by players.
4. **Suspended** — temporarily removed (policy violation, developer
   request).
5. **Delisted** — permanently removed (players with entitlements keep
   access).

## Review process

The review is primarily automated:

- Package validation (schema, signing, size limits).
- Content rating check (age rating matches declared content).
- Malware scan.
- Policy compliance check (no forbidden content).

Human review is triggered for:
- First-time developers.
- High-profile releases.
- Flagged content (reported by users or automated scan).

## API access

The Developer Portal exposes a REST API for CI/CD integration:

```bash
# Upload a new build
curl -X POST https://api.playos.org/dev/v1/apps/:appId/builds \
  -H "Authorization: Bearer $DEV_TOKEN" \
  -F "package=@myapp-1.0.0.gpk" \
  -F "channel=beta"
```

## Related

- [Package Storage](06-package-storage.md)
- [Marketplace Catalog](07-marketplace-catalog.md)
- [Payments and Payouts](10-payments-and-payouts.md)
- [Release Channels](../10-package-format/14-release-channels.md)
