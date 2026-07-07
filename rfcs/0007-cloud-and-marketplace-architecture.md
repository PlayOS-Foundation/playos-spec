---
rfc: 0007
title: Cloud and Marketplace Architecture
status: Accepted
authors: []
created: 2026-01-01
---

# RFC-0007: Cloud and Marketplace Architecture

> **Status:** Accepted

## Summary

PlayOS cloud services and the marketplace are **self-hostable by design**,
**multi-tenant**, and **SDK-first**. This RFC defines the service boundaries,
tenancy model, store-source architecture, and the self-hosting requirement.
The full architecture is specified in Part XI of the PlayOS Book.

## Motivation

The cloud is not the platform — but it must exist for cloud saves,
achievements, leaderboards, and marketplace distribution. Without clear
architecture rules, the platform risks becoming dependent on a single central
provider or fragmenting into incompatible stores. This RFC sets the
architectural rules that keep the ecosystem open.

## Guide-Level Explanation

### Cloud services (self-hostable)

| Service | Purpose | Self-hosted stack (recommended, not mandated) |
|---|---|---|
| Accounts / Identity | Player and developer accounts | Keycloak / Authentik |
| Package Storage | `.gpk` hosting | MinIO / S3-compatible |
| Marketplace Catalog | Store listings, search | PostgreSQL + Meilisearch |
| Cloud Saves | Sync save data across devices | Object storage + metadata in PostgreSQL |
| Leaderboards | Submit scores, query rankings | PostgreSQL |
| Achievements | Unlock tracking, player profile | PostgreSQL |
| Analytics | Telemetry, crash reports | Prometheus + Grafana / Loki |

Every service is a documented, self-hostable component. The hosted PlayOS Hub
is one deployment of these components; a community, school, or OEM can run
their own.

### Marketplace (multi-tenant, SDK-first)

A **tenant** is a store (official, community, OEM, private, LAN). The
marketplace client supports multiple store sources:

```text
Store Sources
 ├── official.playos.dev
 ├── community.example.org
 └── local.lan
```

**SDK-first:** a developer with the PlayOS SDK can publish to a store
without owning a PlayOS Runtime Device.

## Reference-Level Explanation

### Tenancy model
- A tenant has its own namespace of packages, entitlements, and payment
  settings.
- The client discovers available stores from its store-sources
  configuration (provisioned by the runtime or device profile).
- Federated identity (OpenID Connect) allows a single player account across
  stores.

### Self-hosting requirement
- Every service defined by PlayOS MUST have a documented self-hosted
  deployment path.
- The hosted PlayOS Hub uses the same APIs and schemas as the self-hosted
  edition; there is no "enterprise-only" API.
- A self-hosted deployment MUST be able to serve clients without phoning
  home to an official PlayOS server.

### Package flow
```text
Developer uploads .gpk → stored in object storage → manifest in catalog
  → signature verified → client downloads → runtime installs
```

### Cloud save flow
```text
Game calls CloudSave::Sync()
  → local save compressed
  → uploaded to object storage
  → metadata updated in PostgreSQL
  → other device downloads latest on next sync
```

## Drawbacks

- Self-hostable everything increases the initial engineering surface of
  cloud services.
- Multi-tenancy adds complexity to the marketplace catalog and entitlement
  model.
- SDK-first publishing requires a build/package/publish workflow that works
  on Windows/macOS/Linux, not just runtime devices.

These costs are accepted per the "Self-Hostable by Design" principle
(RFC-0001).

## Rationale and Alternatives

- **Single official store only:** Rejected — creates a central dependency
  that contradicts the open-platform principle.
- **No cloud services at all:** Rejected — cloud saves, leaderboards, and
  achievements are expected features of a modern console platform.
- **Third-party cloud only (Steamworks-style):** Rejected — ties the
  platform to an external service with its own terms and pricing.

## Unresolved Questions

- Payment processing (10% commission, payouts) is scoped but not designed
  in detail; it belongs to a future marketplace RFC.
- Federation between stores (sharing entitlements or catalog entries) is a
  future concern.
- The exact CDN/update delivery mechanism is an implementation detail of
  `playos-cloud`.
