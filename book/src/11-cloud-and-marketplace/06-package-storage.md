# Package Storage

The **Package Storage** service stores and serves .gpk package files.
It is an S3-compatible object store with a thin content-delivery
layer.

## Storage backend

The reference implementation uses **MinIO** (S3-compatible). Any
S3-compatible storage works:

- AWS S3
- Cloudflare R2
- MinIO (self-hosted)
- Ceph (self-hosted)
- Local filesystem (development)

## Package lifecycle

1. **Upload** — developer uploads a .gpk through the Developer Portal
   or API.
2. **Validate** — the storage service validates the package signature
   and schema.
3. **Index** — the package is registered in the Marketplace Catalog.
4. **Replicate** — the package is replicated to CDN edge nodes.
5. **Serve** — the Runtime downloads the package from the nearest CDN
   edge.

## Addressing

Packages are addressed by content hash (SHA-256), not by filename:

```
https://cdn.playos.org/pkgs/ab12cd34...ef5678.gpk
```

The content hash prevents tampering: if the file changes, the address
changes. The catalog maps `(appId, version)` to content hash.

## Delta updates

The storage service supports **delta updates** — downloading only the
bytes that changed between versions. This reduces bandwidth for
players and CDN costs for developers. Delta computation uses bsdiff
or similar algorithm.

## Quotas

| Tier | Maximum package size | Total storage |
|---|---|---|
| Free | 2 GB | 10 GB |
| Developer | 10 GB | 100 GB |
| Enterprise | 50 GB | 1 TB |

## Self-hosting

Package storage is a MinIO instance. Self-hosted deployments point
the catalog at their own MinIO. No special configuration is needed.

## Related

- [Marketplace Catalog](07-marketplace-catalog.md)
- [Updates and CDN](16-updates-and-cdn.md)
- [Package Validation](../10-package-format/16-package-validation.md)
- [Signing](../10-package-format/10-signing.md)
