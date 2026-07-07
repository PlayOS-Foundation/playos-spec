# AGENTS.md

Guidance for AI coding agents and human contributors working in the
`playos-spec` repository.

## What this repository is

`playos-spec` is the **source of truth** for the PlayOS platform. It contains the
specification (the PlayOS Book), design proposals (RFCs), accepted decisions
(ADRs), and machine-readable schemas. It does **not** contain platform
implementations — those live in sibling repositories such as
`playos-platform-api`, `playos-runtime`, `playos-shell`, `playos-cloud`,
`playos-marketplace`, `playos-tools`, `playos-samples`, and
`playos-reference-devices`.

## Golden rules

1. **Specification first.** If a change affects platform behavior, update the
   specification (and, when needed, an RFC) here before implementing it in other
   repositories.
2. **One concern per change.** Prefer editing a single chapter, RFC, ADR, or
   schema at a time. The structure is designed so work can be parallelized
   safely.
3. **Keep the contract engine-agnostic.** The Platform API must not depend on
   Raylib or any single engine. Raylib is a reference technology only.
4. **Respect normative language.** Use "must", "should", and "may" as defined in
   `book/src/00-front-matter/04-normative-language.md`.
5. **Do not invent architecture silently.** If a decision is non-trivial, capture
   it in an RFC (`rfcs/`) or ADR (`adr/`).

## Where things go

| Content | Location |
|---|---|
| Readable specification and guides | `book/src/` |
| Table of contents | `book/src/SUMMARY.md` |
| Design proposals | `rfcs/` |
| Accepted decisions | `adr/` |
| Machine-readable formats | `schemas/` |
| Diagrams and branding | `assets/` |

## Editing the book

- Every chapter is a Markdown file referenced from `book/src/SUMMARY.md`.
- When adding a chapter, add both the file and its `SUMMARY.md` entry.
- Build locally with `mdbook build book` (or `mdbook serve book`).

## Adding an RFC

1. Copy `rfcs/0000-template.md` to `rfcs/NNNN-short-title.md`.
2. Use the next available number.
3. Fill in every section.

## Adding an ADR

1. Copy `adr/0000-template.md` to `adr/NNNN-short-title.md`.
2. Set the status (`Proposed`, `Accepted`, `Superseded`, `Deprecated`).

## Style

- Prefer short, declarative sentences.
- Use fenced code blocks for code and ASCII diagrams.
- Keep one Markdown heading level per logical section.
