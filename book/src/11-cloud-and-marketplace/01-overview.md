# Cloud and Marketplace Overview

PlayOS Cloud provides **optional cloud services** for save sync,
leaderboards, achievements, analytics, and application distribution.
Every cloud service is self-hostable.

## Services

| Service | Description | API |
|---|---|---|
| **Accounts** | Player identity, sign-in, friends | `cloud.accounts` |
| **Cloud Saves** | Save file sync across devices | `cloud.saves` |
| **Leaderboards** | Global and friend rankings | `cloud.leaderboards` |
| **Achievements** | Unlock and track achievements | `cloud.achievements` |
| **Marketplace** | Application discovery and purchase | `platform.marketplace` |
| **Analytics** | Anonymous usage data | `cloud.analytics` |
| **Crash Reporting** | Crash dump collection | `cloud.crash-reporting` |
| **Updates** | OTA update distribution | `system.updates` |

## Architecture principle

```
┌─────────────┐     ┌──────────────────┐     ┌──────────────────┐
│  Runtime     │────▶│  PlayOS Cloud    │◀────│  Self-Hosted     │
│  Device      │     │  (Default)       │     │  Instance        │
└─────────────┘     └──────────────────┘     └──────────────────┘
```

The Runtime connects to a single cloud endpoint. That endpoint can be
PlayOS Cloud (the hosted service) or a self-hosted instance. The
Runtime does not care which one it is talking to.

## Chapter index

| Chapter | Description |
|---|---|
| [Self-Hosting Principles](02-self-hosting-principles.md) | How self-hosting works |
| [Cloud Architecture](03-cloud-architecture.md) | Service architecture |
| [Accounts and Identity](04-accounts-and-identity.md) | Player accounts |
| [Developer Portal](05-developer-portal.md) | Developer-facing tools |
| [Package Storage](06-package-storage.md) | Where .gpk files live |
| [Marketplace Catalog](07-marketplace-catalog.md) | App discovery |
| [Store Sources](08-store-sources.md) | Multi-store support |
| [Entitlements](09-entitlements.md) | Ownership verification |
| [Payments and Payouts](10-payments-and-payouts.md) | Monetization |
| [Cloud Saves](11-cloud-saves.md) | Save file sync |
| [Leaderboards](12-leaderboards.md) | Rankings |
| [Achievements](13-achievements.md) | Achievement tracking |
| [Analytics](14-analytics.md) | Usage data |
| [Crash Reporting](15-crash-reporting.md) | Crash collection |
| [Updates and CDN](16-updates-and-cdn.md) | Content delivery |
| [Federation and Private Stores](17-federation-and-private-stores.md) | Inter-store sharing |
| [OEM Stores](18-oem-stores.md) | Hardware-maker stores |
| [Cloud API Versioning](19-cloud-api-versioning.md) | API compatibility |

## Related

- [Cloud API chapters](../06-platform-api/)
- RFC-0007 (Cloud and Marketplace Architecture)
