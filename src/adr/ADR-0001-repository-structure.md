# ADR-0001 — Six-Repository Structure

**Date:** Sprint 0  
**Status:** Accepted  
**Deciders:** PlayOS core team

---

## Context

PlayOS requires multiple distinct components: a PID 1 process supervisor, a Wayland compositor, a shell, a public game API, an internal runtime, and a build system. The question is how to organize this code — monorepo vs separate repositories.

## Decision

Adopt six separate repositories with strict ownership and dependency direction:

| Repository | Owns |
|---|---|
| `playos-spec` | Architecture, contracts, ADRs, schemas, roadmap |
| `playos-platform-api` | Public `libplayos` C ABI |
| `playos-runtime` | Internal IPC and lifecycle transport |
| `playos-compositor` | wlroots compositor |
| `playos-shell` | Raylib controller shell |
| `playos-refdistro` | Buildroot integration and images |

## Rationale

- **Dependency enforcement:** Separate repos make it impossible to accidentally import private internals from `playos-runtime` into a game (the game can only depend on `playos-platform-api`)
- **Independent versioning:** The public API (`playos-platform-api`) can be versioned independently from internal transport changes
- **Clear ownership:** Each repo has a single responsible area; contributors know where their change belongs
- **Parallel work:** Teams can work on compositor and shell independently without merge conflicts

## Consequences

- More overhead for cross-repo changes (PRs in multiple repos must be coordinated)
- `versions.lock` in `playos-refdistro` is required to keep all components pinned
- CI must test integration across repos
