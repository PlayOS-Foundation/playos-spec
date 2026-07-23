# Self-Hosting Principles

Every PlayOS Cloud service **must** be self-hostable. This is a
first-class design constraint, not an afterthought.

## Why self-hosting

- **Privacy** — players keep their data on their own hardware.
- **Independence** — no dependency on PlayOS Cloud staying online
  forever.
- **Offline-first** — LAN parties, air-gapped environments, embedded
  systems.
- **Trust** — users can inspect the server code and verify it does
  what it claims.

## How it works

The Runtime connects to a single cloud endpoint configured at the
system level:

```toml
# /etc/playos/cloud.toml
[cloud]
endpoint = "https://cloud.playos.org"   # Default
# endpoint = "https://my-server.local"  # Self-hosted
```

When the endpoint is set to a self-hosted instance, the Runtime uses
the same API — there is no "cloud mode" vs "self-hosted mode."
The protocol is identical.

## What must be self-hostable

| Service | Self-hostable | Notes |
|---|---|---|
| Accounts | ✅ | OIDC-compatible |
| Cloud Saves | ✅ | S3-compatible backend |
| Leaderboards | ✅ | PostgreSQL backend |
| Achievements | ✅ | PostgreSQL backend |
| Marketplace | ✅ | Static + catalog DB |
| Analytics | ✅ | Prometheus-compatible |
| Crash Reporting | ✅ | Sentry-compatible |
| Package Storage | ✅ | S3-compatible |
| Payments | ❌ | Requires payment processor |

Payments are the only service that cannot be fully self-hosted
(requires a third-party payment processor). All other services can run
on a single Linux server or a Raspberry Pi.

## Deployment

The reference self-hosted deployment uses Docker Compose:

```yaml
# docker-compose.yml (simplified)
services:
  accounts:
    image: playos/accounts:latest
  saves:
    image: playos/saves:latest
  leaderboards:
    image: playos/leaderboards:latest
  achievements:
    image: playos/achievements:latest
  marketplace:
    image: playos/marketplace:latest
  postgres:
    image: postgres:16
  minio:
    image: minio/minio:latest
```

One command: `docker compose up -d`.

## Offline mode

Devices **must** function without a cloud connection. When the cloud
endpoint is unreachable:

- Saves are stored locally and synced when connectivity returns.
- Leaderboards show the last cached rankings.
- Achievements unlock locally and sync later.
- Marketplace is unavailable (applications are already installed).

## Related

- [Self-Hostable by Design](../02-platform-principles/06-self-hostable-by-design.md)
- [Cloud Architecture](03-cloud-architecture.md)
- [Offline and Air-Gap Support](../02-platform-principles/05-runtime-independent.md)
