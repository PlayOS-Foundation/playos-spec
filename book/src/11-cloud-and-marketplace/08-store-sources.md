# Store Sources

PlayOS supports **multiple store sources** — the Marketplace Catalog
can aggregate applications from multiple stores, not just the official
PlayOS Marketplace.

## Store source model

A **store source** is any server that implements the Marketplace
Catalog API. The Runtime can be configured to query multiple sources:

```toml
# /etc/playos/stores.toml
[[stores]]
name = "PlayOS Marketplace"
url = "https://catalog.playos.org"
enabled = true

[[stores]]
name = "My Self-Hosted Store"
url = "https://my-store.local"
enabled = true

[[stores]]
name = "OEM Store"
url = "https://store.oem-hardware.com"
enabled = true
```

## How it works

1. The Marketplace UI queries all enabled store sources.
2. Results are deduplicated by `appId`.
3. If the same application appears in multiple stores, the player
   chooses which store to download from.
4. Entitlements are tracked per-store.

## Use cases

| Use case | Configuration |
|---|---|
| Default | Only PlayOS Marketplace |
| Self-hosted | PlayOS Marketplace + private store |
| OEM device | PlayOS Marketplace + OEM store |
| Enterprise | Private store only |

## Private stores

A private store is a store source that requires authentication. The
Runtime authenticates using the player's account or a device
certificate. Private stores are used for:

- Enterprise deployments
- Education (school-managed devices)
- Developer beta programs

## Federation

When stores **federate**, they share catalog data. A player browsing
OEM Store A can see applications from OEM Store B if the stores agree
to federate. This is opt-in: stores must explicitly enable federation.

See [Federation and Private Stores](17-federation-and-private-stores.md)
for details.

## Related

- [Marketplace Catalog](07-marketplace-catalog.md)
- [Federation and Private Stores](17-federation-and-private-stores.md)
- [OEM Stores](18-oem-stores.md)
- [Self-Hosting Principles](02-self-hosting-principles.md)
