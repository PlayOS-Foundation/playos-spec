# PlayOS Agent Instructions

This repository defines the PlayOS Platform Specification.

Agents working in this repository must follow these rules.

## Core Rules

1. PlayOS is a platform specification, not only an operating system.
2. PlayOS is not a game engine.
3. The PlayOS Platform API must remain engine agnostic.
4. Raylib is the reference engine and reference shell technology, but
   not a hard dependency of the Platform API.
5. Prefer capability checks over platform checks.
6. Distinguish clearly between Runtime Devices and SDK Targets.
7. Do not introduce implementation details unless the specification
   requires them.
8. Do not create APIs that only work on one device unless they are
   explicitly marked as device-specific or vendor extensions.
9. All major design decisions must go through an RFC or ADR.
10. The specification is the source of truth for all other PlayOS
    repositories.

## Terminology

Use these terms consistently:

- PlayOS Platform Specification
- PlayOS Platform API
- PlayOS Runtime
- PlayOS Shell
- PlayOS Cloud
- PlayOS Marketplace
- Runtime Device
- SDK Target
- Capability
- Device Profile
- Reference Implementation

Avoid these mistakes:

- Do not describe PlayOS as just a Linux distribution.
- Do not describe the Platform API as a Raylib extension.
- Do not hardcode ROG Ally, Android, PS Vita, or Orange Pi assumptions
  into the general platform model.
- Do not use platform-specific checks where a capability query should
  be used.

## Documentation Rules

- Book chapters live in `book/src/`.
- RFCs live in `rfcs/`.
- ADRs live in `adr/`.
- Update `book/src/SUMMARY.md` when adding book chapters.
- Keep Markdown clear and readable.
- Prefer short, precise sections.
- Include examples where useful.

## First Principle

Every implementation should follow the specification.

If the specification is unclear, update the specification before
implementing code in other repositories.
