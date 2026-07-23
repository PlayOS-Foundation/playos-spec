# Self-Hostable by Design

Every PlayOS cloud service must be **self-hostable**: a community, school,
OEM, or individual must be able to run their own instance without depending
on an official PlayOS server.

## The principle

> If it requires a cloud service, it must ship with a self-hosted
> deployment path.

The hosted PlayOS Hub (official) is one deployment of the cloud services.
A self-hosted instance is another. They use the same APIs, schemas, and
protocols. There is no "enterprise-only" feature or API that is locked to
the official deployment.

## What must be self-hostable

| Service | Self-hosted stack (recommended) |
|---|---|
| Accounts / Identity | Keycloak, Authentik, or any OIDC provider |
| Package Storage | MinIO, any S3-compatible object store |
| Marketplace Catalog | PostgreSQL + search backend |
| Cloud Saves | Object storage + PostgreSQL |
| Leaderboards | PostgreSQL |
| Achievements | PostgreSQL |
| Analytics | Prometheus + Grafana / Loki |
| Crash Reporting | Sentry (self-hosted) |

## What "self-hostable" means

1. **Documented** — a deployment guide exists for each service.
2. **Containerized** — official Docker/Podman images or a
   `docker-compose.yml` that brings up a complete instance.
3. **No phone-home** — a self-hosted instance must work fully offline
   without contacting an official PlayOS server.
4. **API-compatible** — the self-hosted instance exposes the same
   Cloud API as the official deployment.

## Why this matters

- **Community servers** — a gaming community can run their own store and
  cloud services.
- **OEM devices** — a hardware manufacturer can bundle their own cloud
  without depending on PlayOS Foundation infrastructure.
- **Schools and LANs** — a local network can run PlayOS services for
  classroom or tournament use without internet access.
- **Sovereignty** — no entity can shut down PlayOS by turning off a
  central server.

## Non-goal

Self-hostable does not mean "every player runs their own server." It
means the server software is available and documented for those who
need it.

## Related

- [Open Platform](07-open-platform.md)
- [Cloud Architecture](../11-cloud-and-marketplace/03-cloud-architecture.md)
- [Self-Hosting Principles](../11-cloud-and-marketplace/02-self-hosting-principles.md)
- RFC-0007 (Cloud and Marketplace Architecture)
