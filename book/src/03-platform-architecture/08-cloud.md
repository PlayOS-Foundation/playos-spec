# Cloud

The **PlayOS Cloud** is the hosted service that powers cloud saves,
leaderboards, achievements, accounts, and the marketplace. It is an
optional layer — devices function fully offline.

## Cloud services

| Service | Description |
|---|---|
| Accounts | Player identity, sign-in, friends |
| Cloud Saves | Save file sync across devices |
| Leaderboards | Global and friend rankings |
| Achievements | Achievement tracking and Gamerscore |
| Marketplace | Application discovery and purchase |
| Analytics | Anonymous usage data |
| Crash Reporting | Crash dump collection |
| Package Storage | .gpk hosting |
| Update CDN | OTA update distribution |

## Architecture

The cloud is a collection of microservices communicating over gRPC.
Every service:

- Is stateless (state in databases).
- Exposes health and metrics endpoints.
- Can be deployed and scaled independently.
- Is self-hostable.

## Protocol

Cloud APIs are defined in Protocol Buffers in the
[`playos-cloud-proto`](https://github.com/PlayOS-Foundation/playos-cloud-proto)
repository. The protocol is the contract between the Runtime and the
cloud — any server that implements the protocol is a valid PlayOS Cloud
endpoint.

## Self-hosting

Every cloud service must be self-hostable. The reference self-hosted
deployment uses Docker Compose. See [Self-Hosting Principles](
../11-cloud-and-marketplace/02-self-hosting-principles.md).

## Service boundaries

```
Runtime Device
    │
    │ HTTPS + gRPC
    ▼
API Gateway (TLS, rate limiting, auth)
    │
    ├── Accounts Service ─── PostgreSQL
    ├── Saves Service ────── S3/MinIO
    ├── Leaderboards ─────── PostgreSQL
    ├── Achievements ─────── PostgreSQL
    ├── Marketplace ──────── PostgreSQL
    └── CDN ──────────────── S3/MinIO
```

## Related

- [Cloud and Marketplace](../11-cloud-and-marketplace/01-overview.md)
- [Cloud Architecture](../11-cloud-and-marketplace/03-cloud-architecture.md)
- [Self-Hosting Principles](../11-cloud-and-marketplace/02-self-hosting-principles.md)
- [Self-Hostable by Design](../02-platform-principles/06-self-hostable-by-design.md)
