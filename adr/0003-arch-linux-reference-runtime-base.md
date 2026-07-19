# Use Arch Linux as the Reference Runtime Base

- **ADR:** 0003
- **Title:** Use Arch Linux as the PlayOS Reference Runtime Base
- **Status:** Superseded by [ADR-0004](0004-use-alpine-linux-reference-os-base.md)
- **Date:** 2026-01-01
- **Superseded:** 2026-07-19
- **Deciders:** PlayOS Foundation

## Context

PlayOS Runtime Devices need a Linux operating-system layer beneath the compositor and shell. The original reference implementation selected Arch Linux to obtain a current kernel, Mesa, wlroots, and a reproducible ISO workflow while developing the first ROG Ally vertical slice.

## Decision

PlayOS originally used Arch Linux, `pacman`, `PKGBUILD`, systemd, and `archiso` for the first reference image.

This decision is superseded by [ADR-0004](0004-use-alpine-linux-reference-os-base.md), which selects Alpine Linux as the PlayOS reference OS base.

The Arch implementation remains valuable as history, but its profile and packaging artifacts are no longer active. Git history is the canonical archive. Those artifacts must not define new PlayOS platform behaviour.

## Consequences

The Arch vertical slice proved the PlayOS compositor, shell, GPU path, and ROG Ally bring-up. Distribution-neutral hardware findings and boot measurements should be carried into Alpine documentation and validation; Arch-specific packaging and services should remain in history.

For the original analysis and rationale, consult this file's Git history.
