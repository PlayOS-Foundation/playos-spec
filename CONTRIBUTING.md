# Contributing to playos-spec

Thanks for contributing to the PlayOS specification.

## Before Making Changes

1. Read [`src/architecture.md`](src/architecture.md) — it is the ground truth.
2. Check existing [ADRs](src/adr/) for relevant decisions.
3. File an issue for API changes before editing.

## Adding Documents

1. Create the file in the appropriate `src/` subdirectory.
2. Add it to `src/SUMMARY.md` in the correct position.
3. Link to it from at least one existing document.
4. Run `mdbook build` to verify no broken links.

## Adding ADRs

1. Copy format from an existing ADR.
2. Number sequentially: `ADR-NNNN-short-title.md`.
3. Status: `Proposed` | `Accepted` | `Superseded by ADR-XXXX` | `Deprecated`.
4. Add to `src/SUMMARY.md`.

## Rules

- ADRs are append-only — supersede, never delete.
- Do not commit `book/` — it is in `.gitignore`.
- Do not add implementation code — this is a documentation repository.
- All code blocks must specify the language.

See [`AGENTS.md`](AGENTS.md) for AI agent conventions.
