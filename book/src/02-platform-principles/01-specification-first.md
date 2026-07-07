# Specification First

**Everything in PlayOS is specified before it is implemented.**

The PlayOS Book is the source of truth. Implementation repositories follow
the specification; they do not define platform behavior independently.

## The rule

> If a feature affects platform behavior, it must be specified here before
> it is implemented elsewhere.

This applies to Platform API behavior, runtime behavior, package behavior,
capability behavior, device profile behavior, certification behavior, and
cloud and marketplace behavior.

## Why

PlayOS spans many repositories, devices, engines, and contributors. Without
a single contract, each implementation would drift and the platform would
fragment. A specification-first process keeps everything coherent as it
grows.

## How decisions flow

```text
Idea
  → RFC (for non-trivial changes)
    → Specification update (book/)
      → Accepted decision (adr/)
        → Implementation (other repos)
```

- **RFCs** (`rfcs/`) capture proposals and discussion.
- **ADRs** (`adr/`) record accepted decisions.
- The **book** (`book/src/`) is the readable, authoritative specification.
- **Schemas** (`schemas/`) are the machine-readable contracts.

## Precedence

When sources conflict, the Platform Specification, accepted RFCs, accepted
ADRs, schemas, and conformance requirements take priority over examples or
explanatory text.

## Consequence for implementation repositories

Repositories such as `playos-platform-api`, `playos-runtime`, and
`playos-shell` implement the contract defined here. If the specification is
unclear, the correct action is to **update the specification first**, then
implement.

See also: [Engine Agnostic](02-engine-agnostic.md) and the
[RFC Process](../16-governance-and-process/04-rfc-process.md).
