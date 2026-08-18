# ADR-0005 — RAUC for A/B System Updates

**Date:** Sprint 11  
**Status:** Superseded — resolved to a minimal custom updater (Sprint 11)  
**Deciders:** PlayOS core team

---

## Context

PlayOS needs a signed, atomic A/B update mechanism. Options: RAUC, Mender, SWUpdate, custom EFI-image-specific updater.

## Decision

Use a **minimal custom updater** in `playos-init`, not RAUC. The decision criteria below resolved to custom because the PlayOS system image is a single EFI/squashfs artifact, and the static musl PID 1 cannot carry OpenSSL/PKCS#11.

**Resolution (Sprint 11):** `.playosb` bundle = `[PBS1][LE32 header_len][JSON header][raw squashfs][LE32 sig_len][hex HMAC-SHA256]`, verified with a development HMAC key before any partition write. Production key management (HSM) and dm-verity remain post-MVP.

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
