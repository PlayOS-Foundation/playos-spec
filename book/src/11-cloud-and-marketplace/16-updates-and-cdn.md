# Updates and CDN

The **CDN** (Content Delivery Network) delivers package files and
updates to devices worldwide with low latency.

## CDN architecture

```
Developer Upload → Package Storage → CDN Origin → Edge Nodes
                                                    │
                                          ┌─────────┼─────────┐
                                          ▼         ▼         ▼
                                       Device    Device    Device
```

1. Developer uploads a .gpk to Package Storage.
2. Package Storage notifies CDN of a new package.
3. CDN replicates the package to edge nodes.
4. Devices download from the nearest edge node.

## CDN providers

The reference deployment uses **Cloudflare R2** with Cloudflare CDN.
Self-hosted deployments can use:

- Cloudflare R2 + CDN (default)
- AWS S3 + CloudFront
- MinIO + custom CDN (self-hosted)
- Local file server (single-server deployments)

## Edge caching

Packages are cached at edge nodes with long TTLs (30 days). Package
files are content-addressed (SHA-256), so they are immutable. A new
version has a new hash and a new cache entry. Old versions are evicted
after 90 days.

## Delta updates

The CDN supports **byte-range requests** for delta updates. A device
requests only the changed bytes between versions:

```
GET /pkgs/abc123.gpk
Range: bytes=1048576-2097151
```

The delta is computed server-side using bsdiff and cached alongside
the full package.

## Bandwidth optimization

- Packages are served compressed (gzip/brotli).
- Assets that haven't changed between versions are not re-downloaded.
- Large packages use resumable downloads (HTTP Range requests).

## Update polling

Devices poll for updates at configurable intervals:

- Default: every 6 hours.
- Critical security patches: push notification (when online).
- Developer mode: on-demand (manual check).

Polling uses conditional requests (`If-None-Match`) to avoid
re-downloading catalog data that hasn't changed.

## Self-hosting

Self-hosted deployments configure the CDN endpoint in the cloud
configuration:

```toml
[cloud]
cdnEndpoint = "https://cdn.my-server.local"
```

The CDN endpoint is separate from the catalog API endpoint, allowing
independent scaling.

## Related

- [Package Storage](06-package-storage.md)
- [Updates (Runtime)](../08-runtime-architecture/15-updates.md)
- [Updates and Patches (Package)](../10-package-format/13-updates-and-patches.md)
