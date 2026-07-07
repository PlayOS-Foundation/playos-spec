---
name: RFC Task
about: Create a new RFC or update an existing one.
title: "RFC: "
labels: [rfc, spec]
assignees: []
---

## Goal

<!-- Briefly describe the RFC to create or update. -->

## Context

Read before starting:

- `AGENTS.md`
- `.github/copilot-instructions.md`
- Relevant chapters in `book/src/` (list them below)

## Required Output

<!-- Which files should be created or updated? -->

Create:

- `rfcs/NNNN-short-title.md`

Update if needed:

- `book/src/XX-chapter/XX-page.md`

## Requirements

<!-- Specific topics the RFC must cover. -->

## Rules

- Do not write implementation code.
- Do not hardcode specific platforms in the general API.
- Keep PlayOS engine-agnostic.
- Use capability checks, not platform checks.
