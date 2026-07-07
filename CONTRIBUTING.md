# Contributing to PlayOS

Thank you for your interest in contributing to the PlayOS Platform Specification.

## Ways to contribute

- Improve or write chapters of the PlayOS Book (`book/src/`).
- Propose platform changes through an RFC (`rfcs/`).
- Record accepted decisions as ADRs (`adr/`).
- Refine machine-readable schemas (`schemas/`).
- Report issues, ambiguities, or inconsistencies in the specification.

## Specification-first workflow

PlayOS is specification-first. Changes that affect platform behavior should be
described here before they are implemented in other repositories.

```text
Idea
  → RFC (for non-trivial changes)
    → Specification update (book/)
      → Accepted decision (adr/)
        → Implementation (other repos)
```

## Making changes

1. Create a branch for your change.
2. Keep each pull request focused on a single concern (one chapter, one RFC, one
   ADR, or one schema where possible).
3. If you add a book chapter, update `book/src/SUMMARY.md`.
4. Build the book locally to check formatting:

   ```sh
   mdbook build book
   ```

5. Open a pull request describing the motivation and impact.

## Writing an RFC

1. Copy `rfcs/0000-template.md` to `rfcs/NNNN-short-title.md`.
2. Use the next free number.
3. Complete each section and open a pull request for discussion.

## Normative language

When writing specification text, follow the conventions in
[`book/src/00-front-matter/04-normative-language.md`](book/src/00-front-matter/04-normative-language.md)
for the words "must", "should", and "may".

## Code of conduct

All participation is governed by our [Code of Conduct](CODE_OF_CONDUCT.md).
