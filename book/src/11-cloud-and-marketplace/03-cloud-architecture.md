# Cloud Architecture

PlayOS Cloud is a collection of **independent microservices** that
communicate over HTTP/2 and gRPC. Each service can be deployed,
scaled, and self-hosted independently.

## Service topology

```
                     ┌──────────────────┐
                     │   API Gateway    │  (TLS termination, rate limiting)
                     └────────┬─────────┘
          ┌───────────────────┼───────────────────┐
          │                   │                   │
    ┌─────▼─────┐       ┌─────▼─────┐       ┌─────▼─────┐
    │  Accounts │       │  Saves    │       │Leaderboards│
    │  Service  │       │  Service  │       │  Service   │
    └─────┬─────┘       └─────┬─────┘       └─────┬─────┘
          │                   │                   │
    ┌─────▼─────┐       ┌─────▼─────┐       ┌─────▼─────┐
    │ PostgreSQL│       │  S3/MinIO │       │ PostgreSQL │
    └───────────┘       └───────────┘       └───────────┘
```

## Communication

- **Runtime ↔ Cloud**: HTTP/2 REST + gRPC. TLS 1.3 required.
- **Service ↔ Service**: gRPC. Internal network only.
- **CDN ↔ Runtime**: HTTPS. Static assets (packages, images).

## Protocol

All cloud APIs are defined in Protocol Buffers (`.proto` files) in the
[`playos-cloud-proto`](https://github.com/PlayOS-Foundation/playos-cloud-proto)
repository. This is the single source of truth for the cloud API
contract, separate from the Platform API.

## Statelessness

All cloud services are **stateless**. State is stored in databases
(PostgreSQL, S3/MinIO). Services can be scaled horizontally behind a
load balancer. Session state, if needed, is stored in the database,
not in memory.

## Authentication

The API Gateway authenticates requests using **OAuth 2.0 / OIDC**
bearer tokens. Internal service-to-service calls use mutual TLS
(mTLS). The Runtime authenticates with a device certificate
issued at first boot.

## Observability

Every service exposes:

- **Health check** — `GET /health` (liveness) and `GET /ready`
  (readiness).
- **Metrics** — Prometheus endpoint on `/metrics`.
- **Structured logging** — JSON logs to stdout.

## Reference implementation

The reference implementation is written in Rust (Actix-web for HTTP,
Tonic for gRPC). Alternative implementations in other languages are
acceptable as long as they conform to the `.proto` contracts.

## Related

- [Cloud Saves](11-cloud-saves.md)
- [Accounts and Identity](04-accounts-and-identity.md)
- [Self-Hosting Principles](02-self-hosting-principles.md)
- RFC-0007 (Cloud and Marketplace Architecture)
