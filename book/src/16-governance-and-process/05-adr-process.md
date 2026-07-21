# ADR Process

Architecture Decision Records (ADRs) capture significant PlayOS architecture decisions together with their context, consequences, and considered alternatives.

ADRs complement the PlayOS Book and RFC process:

- the **PlayOS Book** explains the coherent platform architecture and developer model;
- an **ADR** records why a durable architecture decision was made;
- an **RFC** proposes substantial new behavior, contracts, or ecosystem changes before adoption.

## When an ADR is required

An ADR should be created when a decision:

- affects multiple PlayOS repositories or platform layers;
- establishes a long-lived technology, dependency, or architectural boundary;
- changes a previously accepted architecture decision;
- defines a reference implementation strategy with ecosystem consequences;
- would otherwise be difficult to reconstruct from code or discussion history.

Routine implementation details, local refactors, and easily reversible choices generally do not require an ADR.

## Location and numbering

ADRs are stored in the repository-level `adr/` directory.

File names use a zero-padded sequence and descriptive slug:

```text
adr/0005-playos-typography-and-text-rendering-strategy.md
```

Numbers are never reused, even when an ADR is rejected or superseded.

## Status lifecycle

An ADR normally moves through these states:

- **Proposed** — under discussion and not yet authoritative;
- **Accepted** — approved and normative for the scope it defines;
- **Rejected** — considered but not adopted;
- **Deprecated** — retained for history but no longer recommended;
- **Superseded** — replaced by a newer ADR, which must be linked explicitly.

Accepted ADRs are not silently rewritten to represent a different decision. Material changes require either a clearly documented amendment or a superseding ADR.

## Required content

An ADR should contain:

- title, number, status, date, and deciders;
- context and the problem being resolved;
- the decision and its architectural boundaries;
- positive and negative consequences;
- alternatives considered;
- risks, mitigations, and non-goals where relevant;
- links to related ADRs, RFCs, and PlayOS Book chapters.

The repository ADR template should be used as the starting point.

## Relationship to the PlayOS Book

The PlayOS Book should explain accepted architecture in a coherent, implementation-facing form. It should link to the relevant ADR as the normative record of the decision.

The ADR should not become a duplicate copy of an entire Book chapter. Instead:

- the ADR records **why** the decision was made and what alternatives were rejected;
- the Book explains **how the accepted architecture fits together** and what implementers must understand.

When an accepted ADR changes the architecture, the relevant Book chapters and navigation should be updated in the same pull request whenever practical.

## Review and acceptance

ADR pull requests should be reviewed for:

- consistency with existing accepted decisions;
- clear ownership boundaries between platform layers;
- portability across Runtime Devices and SDK Targets where applicable;
- security, accessibility, compatibility, and operational consequences;
- effects on reference implementations and downstream adopters;
- whether an RFC is also required before acceptance.

The PlayOS Foundation or delegated maintainers determine acceptance according to the governance model.

## Current example

[ADR-0005: PlayOS Typography and Text Rendering Strategy](../../../adr/0005-playos-typography-and-text-rendering-strategy.md) is an accepted decision that defines the reference font stack, text-rendering ownership boundaries, device-profile scaling, accessibility expectations, and internationalization roadmap.

The corresponding Book chapter, [Typography and Text Rendering](../09-shell-and-ux/17-typography-and-text-rendering.md), translates that decision into guidance for the reference shell and runtime architecture.
