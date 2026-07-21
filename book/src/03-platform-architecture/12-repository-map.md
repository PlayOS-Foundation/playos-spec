# Repository Map

The PlayOS platform is split across ten repositories under the
[PlayOS Foundation](https://github.com/PlayOS-Foundation) GitHub organization.
This chapter documents what belongs in each, so contributors and AI agents
know where to make a change.

## The rule

> If a document defines what PlayOS is or how PlayOS must behave, it goes in
> `playos-spec`. If a document explains how to build, run, test, or contribute
> to one implementation, it goes in that implementation's repository.

## Repositories

| Repository | Role | Contains |
|---|---|---|
| [`playos-spec`](https://github.com/PlayOS-Foundation/playos-spec) | **Source of truth.** The PlayOS Book, RFCs, ADRs, and machine-readable schemas. No implementation code. | `book/`, `rfcs/`, `adr/`, `schemas/`, `assets/` |
| [`playos-platform-api`](https://github.com/PlayOS-Foundation/playos-platform-api) | Reference implementation of the engine-agnostic Platform API. | `include/playos/` (public headers), `src/` + `src/backends/` (implementation + platform backends) |
| [`playos-runtime`](https://github.com/PlayOS-Foundation/playos-runtime) | Reference runtime: process launcher, wlroots compositor, runtime services, OS integration. | `include/playos/runtime/`, `src/process*.cpp`, `src/main.cpp` (CLI), `compositor/` (Linux-only) |
| [`playos-shell`](https://github.com/PlayOS-Foundation/playos-shell) | Reference console UI built with Raylib. Controller-first, replaceable. Contains the standalone installer GUI (`playos-installer-gui`). | `src/main.cpp`, `src/installer/` (installer GUI), `CMakeLists.txt` |
| [`playos-samples`](https://github.com/PlayOS-Foundation/playos-samples) | Official sample applications and tutorials. | `hello-playos/` and future samples |
| [`playos-cloud`](https://github.com/PlayOS-Foundation/playos-cloud) | Self-hostable backend services: accounts, cloud saves, achievements, leaderboards, analytics. | Service code, deployment manifests, API definitions |
| [`playos-marketplace`](https://github.com/PlayOS-Foundation/playos-marketplace) | Marketplace and distribution: publishing, discovery, installation, updates. | Marketplace services and client integration |
| [`playos-tools`](https://github.com/PlayOS-Foundation/playos-tools) | Developer tools: packaging, project templates, build utilities, diagnostics, SDK tooling. | CLIs, templates, build/deploy helpers |
| [`playos-reference-devices`](https://github.com/PlayOS-Foundation/playos-reference-devices) | Hardware profiles, platform backends, bring-up guides for supported devices. | Device folders (`rog-ally/`), setup/build/launch scripts, device profiles |
| [`playos-refdistro`](https://github.com/PlayOS-Foundation/playos-refdistro) | Reference OS distribution build system. Alpine Linux image builder, OpenRC services, ISO/PXE tooling, standalone installer backend. | `alpine/` (init scripts, apkovl, mkimage), `scripts/`, `docs/` |
| [`playos-foundation`](https://github.com/PlayOS-Foundation/playos-foundation) | Governance, public website, community resources, branding. No platform code. | Website source, governance docs, brand assets |

## Dependency graph

```text
playos-spec           ← every repo reads this; no repo depends on another
playos-platform-api   ← playos-shell, playos-samples, and games depend on it
playos-runtime        ← playos-shell depends on it for LaunchAndWait
playos-shell          ← depends on platform-api + runtime
playos-samples        ← depends on platform-api
playos-cloud          ← independent; implements spec contracts
playos-marketplace    ← independent; implements spec contracts
playos-tools          ← independent; produces spec-conformant artifacts
playos-reference-devices  ← independent; operational material for hardware
playos-refdistro      ← independent; produces OS images consumed by reference devices
playos-foundation     ← independent; organizational, no code
```

**Implementation repositories must not define platform behavior
independently.** New platform behavior should be specified in `playos-spec`
first, usually through the book, an RFC, or an ADR.
