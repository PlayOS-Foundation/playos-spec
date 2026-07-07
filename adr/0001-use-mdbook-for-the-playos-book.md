# Use mdBook for the PlayOS Book

- **ADR:** 0001
- **Title:** Use mdBook for the PlayOS Book
- **Status:** Accepted
- **Date:** 2026-01-01
- **Deciders:** PlayOS Foundation

## Context

The PlayOS Platform Specification is large, structured into many chapters, and
must be readable as a book, searchable, versionable in Git, and easy for
contributors and AI coding agents to edit one file at a time.

We need a documentation tool that:

- renders Markdown into a navigable, searchable website,
- keeps content as plain Markdown files in the repository,
- supports a clear table of contents (`SUMMARY.md`),
- is simple to build in CI and publish to static hosting.

## Decision

The PlayOS Book is authored and built with [mdBook](https://rust-lang.github.io/mdBook/).

- Book sources live in `book/src/`.
- The table of contents is defined in `book/src/SUMMARY.md`.
- Book configuration lives in `book/book.toml`.

## Consequences

- Content stays as plain Markdown, making it easy to review in pull requests and
  edit with automated tooling.
- Built-in search and theming come for free.
- Contributors need the `mdbook` binary (or the CI pipeline) to build locally.
- Non-linear or highly interactive documentation formats are not available
  without additional mdBook preprocessors or plugins.

## Alternatives Considered

- **Docusaurus / MkDocs** — more features, but heavier toolchains (Node or Python)
  and more configuration than needed for a specification book.
- **Plain Markdown in the repo with no site generator** — simplest, but loses
  navigation, search, and a cohesive "book" reading experience.
- **A single large document** — unmanageable for a specification of this size and
  hard for multiple contributors or agents to edit concurrently.
