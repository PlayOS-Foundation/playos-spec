# ADR-0005 — RAUC for A/B System Updates

**Date:** Sprint 11  
**Status:** Accepted (pending evaluation — may be revised to a simpler custom solution)  
**Deciders:** PlayOS core team

---

## Context

PlayOS needs a signed, atomic A/B update mechanism. Options: RAUC, Mender, SWUpdate, custom EFI-image-specific updater.

## Decision

Use RAUC as the first update engine. Evaluate whether a simpler custom EFI-image-based approach is sufficient before committing to full RAUC integration.

## Rationale

**Why RAUC:**
- Designed for A/B embedded Linux updates — good conceptual match
- Supports signed bundles (OpenSSL/PKCS#11), boot slot management, and rollback
- Buildroot has a `rauc` package
- Used in production by several embedded Linux projects

**Why evaluate a custom approach:**
- PlayOS uses a simple EFI artifact model (not a traditional rootfs tarball)
- RAUC's slot model may require a custom handler for EFI-image slots
- A custom updater that just: verifies a signature, writes to the inactive EFI slot, updates `boot.json`, and reboots might be simpler and smaller
- Less code = smaller attack surface in the update path

## Decision Criteria for Custom vs RAUC

Prefer RAUC if:
- RAUC can handle EFI-image slots with minimal custom handler code
- The bundle signature infrastructure (key management) integrates cleanly with PlayOS signing

Prefer a custom updater if:
- RAUC requires more than 200 lines of custom handler code for EFI slots
- The total update binary size for RAUC exceeds 2 MB in the initramfs

## Alternatives Considered

| Option | Notes |
|---|---|
| Mender | SaaS-oriented; heavier client; less embedded-only focused |
| SWUpdate | Good alternative to RAUC; very similar feature set |
| Custom | Simplest for EFI-image model; no external dependencies; more code to maintain |

## Consequences

- RAUC or the custom updater must be integrated into `playos-refdistro` and tested for full A/B cycle
- Update bundle signing key management must be designed before the first production release
- The choice must be finalized before Sprint 14 (production readiness)
