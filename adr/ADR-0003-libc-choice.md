# ADR-0003 — musl libc Only

**Date:** Sprint 0  
**Status:** Accepted  
**Deciders:** PlayOS core team

---

## Context

The system needs a C library. Options: glibc, musl, uClibc-ng. The choice affects binary size, compatibility, and which packages can be built.

## Decision

Use musl libc exclusively. No glibc support is planned for v1.

## Rationale

- **Size:** musl produces significantly smaller binaries than glibc — critical for an embedded initramfs
- **Static linking:** musl supports full static linking with predictable behavior; glibc static linking has well-known edge cases (NSS, DNS)
- **Reproducibility:** musl is simpler and more deterministic; easier to reason about at the ABI level
- **Buildroot support:** Buildroot's musl toolchain is mature and widely used
- **Security:** Smaller attack surface; musl's allocator is not susceptible to some classic glibc heap exploits

## Alternatives Considered

| Option | Rejected because |
|---|---|
| glibc | Larger; more complex; static linking edge cases; overkill for a console OS |
| uClibc-ng | Less maintained; fewer compatible packages; musl is the better modern choice |

## Consequences

- Some packages assume glibc internals and may require patches (Mesa, wlroots are well-tested with musl)
- The public `libplayos` C ABI must be compatible with musl — no glibc-specific extensions
- The `PLAYOS_API_VERSION` compatibility policy must note the musl dependency
- Games targeting PlayOS must link against musl or be statically linked
