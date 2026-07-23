---
rfc: 0016
title: Self-Hosting and Store Federation
status: Draft
authors: []
created: 2026-07-22
---

# RFC-0016: Self-Hosting and Store Federation

> **Status:** Draft

## Summary

This RFC defines the self-hosting architecture for PlayOS Cloud
services and the store federation protocol that allows multiple
independent stores to interoperate.  It builds on RFC-0007 (Cloud and
Marketplace Architecture) by specifying the deployment model, the
federation manifest format, private and enterprise store
configuration, OEM store integration, and store discovery mechanisms.

## Motivation

RFC-0007 defines the cloud architecture at a high level.  This RFC
answers the practical questions:

- How does a community or enterprise deploy their own PlayOS Cloud?
- How do independent stores share catalog data without a central
  authority?
- How does an OEM ship a device with their own store plus the public
  marketplace?
- How does a player discover and add new stores?

Self-hosting is a first-class design constraint (RFC-0001, Principle 6).
This RFC makes it concrete.

## Guide-Level Explanation

PlayOS Cloud is designed to run on anything from a Raspberry Pi to a
cloud datacenter.  One `docker compose up -d` command starts all
services.

Stores can **federate** — agree to share catalog data.  If Store A
and Store B federate, a player browsing Store A sees apps from Store B
too.  This is how independent stores build a shared ecosystem without
a central marketplace monopoly.

OEMs can ship devices with their own store pre-configured, plus the
public marketplace.  Enterprises can run private stores for internal
apps.  It all uses the same protocol.

## Reference-Level Explanation

### 1. Self-Hosting Deployment

#### Reference architecture

```yaml
# docker-compose.yml
services:
  # Core services
  accounts:
    image: playos/accounts:1.0
    environment:
      - OIDC_ISSUER=https://auth.my-server.local
    depends_on: [postgres]
  
  saves:
    image: playos/saves:1.0
    environment:
      - STORAGE_BACKEND=s3
      - S3_ENDPOINT=http://minio:9000
    depends_on: [minio]
  
  leaderboards:
    image: playos/leaderboards:1.0
    depends_on: [postgres]
  
  achievements:
    image: playos/achievements:1.0
    depends_on: [postgres]
  
  marketplace:
    image: playos/marketplace:1.0
    environment:
      - CATALOG_BACKEND=postgres
      - PACKAGE_STORAGE=s3
    depends_on: [postgres, minio]
  
  # Infrastructure
  postgres:
    image: postgres:16-alpine
    volumes: [pgdata:/var/lib/postgresql/data]
  
  minio:
    image: minio/minio:latest
    volumes: [miniodata:/data]
  
  # Optional
  cdn:
    image: playos/cdn:1.0  # or nginx for simple deployments
    depends_on: [minio]
  
  analytics:
    image: playos/analytics:1.0  # Prometheus + Grafana
    depends_on: [postgres]
  
  crashreport:
    image: playos/crashreport:1.0  # Sentry-compatible
    depends_on: [postgres]

volumes:
  pgdata:
  miniodata:
```

#### Hardware requirements

| Scale | Hardware | Services |
|---|---|---|
| **Personal** | Raspberry Pi 4 (4 GB) | All core services |
| **Community** | Single x86 server (8 GB) | Core + CDN + analytics |
| **Enterprise** | Kubernetes cluster | All services, HA |

#### Configuration

The Runtime connects to a single cloud endpoint:

```toml
# /etc/playos/cloud.toml
[cloud]
endpoint = "https://cloud.playos.org"   # Default
# endpoint = "https://my-server.local"  # Self-hosted
# endpoint = "https://store.oem.com"    # OEM store
```

The protocol is identical regardless of the endpoint.  There is no
"cloud mode" vs. "self-hosted mode" — the Runtime uses the same API.

#### Service matrix

| Service | Required | Self-Hostable | Backend Options |
|---|---|---|---|
| Accounts (OIDC) | ✅ | ✅ | Keycloak, Authentik, Dex |
| Cloud Saves | ✅ | ✅ | S3, MinIO, local FS |
| Leaderboards | Optional | ✅ | PostgreSQL |
| Achievements | Optional | ✅ | PostgreSQL |
| Marketplace Catalog | ✅ | ✅ | PostgreSQL + static files |
| Package Storage | ✅ | ✅ | S3, MinIO, local FS |
| CDN | Recommended | ✅ | Cloudflare R2, nginx, MinIO |
| Analytics | Optional | ✅ | Prometheus + Grafana |
| Crash Reporting | Optional | ✅ | Sentry-compatible |
| Payments | Optional | ❌ | Requires Stripe/Paddle |

Payments require a third-party processor and cannot be fully
self-hosted.  All other services can run on a single server.

### 2. Store Federation

#### Federation model

Federation is **mutual** and **non-transitive**:

- Store A and Store B both agree to federate.
- Store A's catalog includes Store B's apps.
- Store B's catalog includes Store A's apps.
- Store C that federates with Store B does NOT automatically get
  Store A's apps.

```text
Store A ←→ Store B ←→ Store C
  ↕                   (no direct link)
Store B's apps         Store B's apps only
+ Store A's apps       + Store C's apps
```

#### Federation manifest

A store publishes its federation configuration at
`/.well-known/playos-federation.json`:

```json
{
  "storeId": "0193a8b2-7c5e-4d1f-9a3b-8c7d6e5f4a3b",
  "storeName": "Indie Games Store",
  "storeEndpoint": "https://store.indiegames.example.com",
  "publicKey": "MCowBQYDK2VwAyEA...",
  "federatedStores": [
    "0193c4d5-6e7f-8a9b-0c1d-2e3f4a5b6c7d"
  ],
  "sharedCategories": ["games"],
  "excludedApps": [],
  "federationPolicy": {
    "requireSignatureVerification": true,
    "requireEntitlementCheck": true,
    "maxCatalogSize": 10000
  }
}
```

#### Federation flow

1. Store A administrator adds Store B's store ID to federation list.
2. Store A fetches Store B's `federation.json`.
3. Store A verifies Store B's manifest signature.
4. Store A fetches Store B's catalog.
5. Store A merges Store B's catalog entries into its own catalog index.
6. Players browsing Store A see Store B's apps with a "from Store B"
   badge.
7. When a player purchases a Store B app through Store A, Store A
   forwards the entitlement to Store B.

#### Entitlement forwarding

When a purchase happens through a federated store:

1. Player buys App X from Store A (federated from Store B).
2. Store A creates an entitlement: `(playerId, appId, source: "federated")`.
3. Store A forwards the entitlement to Store B via API.
4. Store B records the entitlement in its own database.
5. The player can install App X from either store.

### 3. Private and Enterprise Stores

#### Private store configuration

A private store restricts catalog access:

```yaml
# Enterprise store configuration
store:
  type: private
  auth: oidc
  idp: https://idp.company.com
  allowedDomains: ["company.com"]
  allowedEmails: []  # or explicit allowlist
  
  federation:
    - storeId: "playos-marketplace"
      sharedCategories: ["utilities"]
```

#### Access control methods

| Method | Use Case |
|---|---|
| OIDC domain restriction | Enterprise — employees with @company.com |
| OIDC group membership | Department-specific access |
| Client certificate | Kiosk/devices in a physical location |
| IP allowlist | On-premises deployment |
| Invite code | Closed beta, early access |

#### Enterprise deployment

An enterprise deploys a private store for internal applications:

```text
Enterprise Cloud
├── Private Store (internal apps only)
├── Federated with PlayOS Marketplace (public utility apps)
└── OIDC identity provider (company SSO)

Employee device
├── Configured with enterprise cloud endpoint
├── Sees internal apps + federated public apps
└── Authenticated via company SSO
```

### 4. OEM Store Integration

An OEM (device manufacturer) can ship a device with:

1. **Their own store** — pre-configured as the primary store.
2. **Federation with public marketplace** — players also see public apps.
3. **OEM-exclusive apps** — apps available only on that OEM's devices
   (via capability `vendor.<oem>.device`).

OEM store configuration:

```yaml
# OEM device profile includes store configuration
[store]
primary = "https://store.oem.com"
federated = ["https://marketplace.playos.org"]
oemApps = ["oem-dashboard", "oem-game-launcher"]
```

### 5. Store Discovery

Devices discover stores through:

| Method | Mechanism |
|---|---|
| **Pre-configured** | Device profile or OEM image |
| **Manual entry** | Player enters a store URL in Settings |
| **QR code** | Scan a QR code to add a store (handheld-friendly) |
| **Local network** | mDNS/DNS-SD advertisement (`_playos-store._tcp`) |
| **Federation discovery** | Following federation links from a known store |

#### QR code format

```
playos+store://store.example.com
```

The Shell's store settings screen includes a "Scan QR Code" option
that opens the device camera and parses this URI.

#### Federation graph discovery

When a player adds Store A, the Shell queries Store A's federation
manifest and offers to add federated stores:

```
┌──────────────────────────────────────────┐
│  Store Added: Indie Games Store          │
│                                          │
│  This store federates with:              │
│  • PlayOS Marketplace (public)           │
│  • Retro Game Store                      │
│                                          │
│  [Add PlayOS Marketplace]                │
│  [Add Retro Game Store]                  │
│  [Skip]                                  │
└──────────────────────────────────────────┘
```

### 6. Offline and Air-Gap Operation

Devices MUST function without a cloud connection:

| Feature | Online | Offline |
|---|---|---|
| App launching | ✅ (entitlement check) | ✅ (cached entitlements, 30 days) |
| Cloud saves | ✅ (sync) | ✅ (local only, sync later) |
| Leaderboards | ✅ (live) | ✅ (cached rankings) |
| Achievements | ✅ (sync) | ✅ (unlock locally, sync later) |
| Marketplace | ✅ (browse, buy) | ❌ (apps already installed work) |
| Store discovery | ✅ (federation) | ❌ (requires network) |

The entitlement cache expires after 30 days offline.  After that, the
device must reconnect to refresh.

## Drawbacks

- Federation requires store operators to trust each other's entitlement
  records — a malicious store could fabricate entitlements.
- Self-hosting places a burden on the operator to maintain
  infrastructure (updates, backups, security patches).
- Offline entitlement caching creates a window where revoked
  entitlements still grant access (up to 30 days).
- Federation is non-transitive, which limits organic discovery but
  prevents unbounded catalog growth.

## Rationale and Alternatives

- **Central marketplace only (no federation):** Simpler but creates a
  single point of control — violates the Self-Hostable by Design
  principle.  Rejected.
- **Transitive federation (A→B→C means A→C):** Would create a global
  catalog automatically but makes entitlement tracking and revenue
  splitting exponentially complex.  Non-transitive is simpler and
  respects store autonomy.
- **Blockchain-based entitlement tracking:** Decentralized but
  introduces unacceptable complexity, latency, and energy cost for
  a feature that works fine with federated trust.
- **No offline mode:** Console-like experience but fails on handheld
  devices that are frequently offline (commutes, travel, air-gapped
  environments).  Offline-first is essential.

## Prior Art

- **ActivityPub / Fediverse:** The federation model (mutual agreement,
  non-transitive, shared protocol) is directly inspired by
  ActivityPub's model for social networks.
- **Mastodon instance federation:** Store A federating with Store B is
  analogous to Mastodon instance A federating with instance B — users
  on either see content from both.
- **Docker Compose for self-hosting:** The reference deployment model
  follows the pattern used by Sentry, GitLab, and Plausible for
  one-command self-hosted deployments.
- **Steam family sharing:** Entitlement sharing across accounts is
  similar to PlayOS family sharing (defined in RFC-0007).
- **iOS/Android enterprise app distribution:** Private stores with OIDC
  auth follow the same model as Apple Business Manager and Android
  Managed Play.

## Unresolved Questions

- How are revenue splits handled for federated purchases?  Store A
  selling Store B's app — who gets what percentage?  Proposed: Store A
  takes a referral fee (10–15%), remainder goes to Store B and the
  developer per Store B's split.
- How are store signing keys distributed and verified?  Each store has
  its own Ed25519 key; the federation manifest includes the public key.
  Cross-signing or a web of trust may be needed.
- Should there be a maximum federation depth?  Non-transitive federation
  means depth is effectively 1, but stores could form chains.
- How are federated catalog conflicts resolved?  Two stores both claim
  to distribute App X — which one takes precedence?

## Future Possibilities

- **Store ratings and trust scores:** A reputation system for stores in
  the federation, helping players avoid low-quality or malicious stores.
- **Decentralized store discovery:** A DHT-based discovery mechanism so
  stores can be found without a central registry.
- **Cross-store achievements and leaderboards:** Federated achievement
  and leaderboard data so players can compete across stores.
- **Store migration:** Tooling for developers to migrate their app and
  entitlements from one store to another without player impact.
