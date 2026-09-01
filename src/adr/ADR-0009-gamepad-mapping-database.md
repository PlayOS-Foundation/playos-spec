# ADR-0009: Gamepad Identifier Database (SDL GameControllerDB)

Status: Accepted (Sprint 13.6)

## Context

The ROG Ally internal controller presents as "Microsoft X-Box 360 pad" with
Linux aliasing that made physical X/Y buttons behave swapped in games, the
controller visualizer, and the shell. We needed a way for all known
controllers to map correctly without per-device special-casing everywhere.

## Decision

- Embed the **SDL GameControllerDB** Linux mappings in `playos-platform-api`
  (game path) and `playos-shell` (trusted shell path, vendor copy).
- Use SDL-style GUIDs from `EVIOCGID` and SDL's enumerated index semantics
  (buttons from `BTN_MISC`, hats even-only, axes increasing code order).
- Resolution precedence: device quirk → DB entry → Xbox-standard fallback.
- The ROG Ally face-swap quirk is applied only on the fallback path; when a DB
  entry matches, the DB mapping already encodes correct X/Y and the quirk is
  disabled to avoid double-swapping.

## Consequences

- New controllers are supported by adding DB lines, not code.
- Two copies of the DB exist (game + shell) to preserve the shell trust
  boundary; they are kept in sync.
- Shell/game mapping semantics are now consistent.
