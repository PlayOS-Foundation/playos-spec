# AGENTS.md — playos-spec

This repository is the **single source of truth** for the PlayOS project. It contains architecture documents, component specifications, API contracts, sprint plans, and Architecture Decision Records (ADRs). It is structured as an [mdBook](https://rust-lang.github.io/mdBook/).

## Repository Layout

```
src/
├── SUMMARY.md                  ← Navigation tree — update when adding docs
├── README.md                   ← Landing page / project intro
├── architecture.md             ← Master system design (read this first)
├── platform-api.md             ← libplayos C ABI specification
├── runtime-ipc.md              ← Internal IPC protocol
├── wayland-protocol.md         ← Private Wayland protocol
├── security-model.md           ← Trust zones, seccomp, Landlock
├── playos-init-spec.md         ← PID 1 supervisor spec
├── playos-compositor-spec.md   ← wlroots compositor spec
├── playos-shell-spec.md        ← Raylib shell spec
├── playos-overlay-spec.md      ← System overlay spec
├── build-guide.md              ← Buildroot / br2-external guide
├── kernel-config.md            ← Kernel kconfig reference
├── dev-environment.md          ← QEMU dev setup, USB boot
├── testing.md                  ← 7-layer CI strategy
├── post-mvp.md                 ← Post-MVP feature backlog
├── sprints/
│   ├── roadmap.md              ← MVP criteria and sprint table
│   └── Sprint-0.md … Sprint-19.md
└── adr/
    └── ADR-0001.md … ADR-0008.md
```

## What to Read Before Making Changes

- **Any change**: read `src/architecture.md` — it is the ground truth for all design decisions.
- **API changes**: read `src/platform-api.md` and file an issue before editing.
- **New ADR**: read existing ADRs in `src/adr/` for format reference. ADRs are append-only — never edit a superseded ADR's decision; instead create a new one that supersedes it.
- **Sprint changes**: read `src/sprints/roadmap.md` first; sprint docs reference each other's exit gates.

## Conventions

- **Language**: British/American English, present tense, imperative mood for headings.
- **Code blocks**: always specify the language (` ```c `, ` ```xml `, ` ```sh `).
- **Tables**: use GFM pipe tables. Align columns for readability.
- **Cross-links**: use relative paths, e.g. `[architecture](architecture.md)`.
- **Section IDs**: do not rename headings without checking that no other doc links to them via anchor.
- **SUMMARY.md**: must be updated whenever a doc is added, removed, or renamed. mdBook will fail to build if SUMMARY.md references a missing file.

## Adding a New Document

1. Create the file in the appropriate `src/` subdirectory.
2. Add it to `src/SUMMARY.md` in the correct position.
3. Link to it from at least one existing document.
4. Run `mdbook build` to verify no broken links.

## Adding a New ADR

1. Copy the format from an existing ADR (`src/adr/ADR-0001-*.md`).
2. Number sequentially: `ADR-NNNN-short-title.md`.
3. Status must be one of: `Proposed` | `Accepted` | `Superseded by ADR-XXXX` | `Deprecated`.
4. Add to `src/SUMMARY.md` under the ADR section.

## Build Command

```sh
mdbook build      # outputs to book/
mdbook serve      # live-reload preview at http://localhost:3000
```

## What NOT to Do

- Do not delete or rewrite ADRs — supersede them.
- Do not commit `book/` — it is in `.gitignore` and built by CI.
- Do not change `src/ideas.md` — it is the original source document, kept for historical reference only.
- Do not add implementation code here — code lives in the other PlayOS repositories.
