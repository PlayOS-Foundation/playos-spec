# PlayOS Specification

The **single source of truth** for the PlayOS gaming console operating system.

## What's Here

- **Architecture** — system design, process model, boot sequence, security boundaries
- **Component specifications** — init, compositor, shell, overlay, platform API
- **API contracts** — `libplayos` public C ABI, internal IPC protocol, Wayland protocol
- **ADR** — Architecture Decision Records (append-only)
- **Sprints** — MVP roadmap and sprint-by-sprint implementation plans
- **Guides** — build, dev environment, kernel config, testing

## Getting Started

Read [`src/architecture.md`](src/architecture.md) first — it is the ground truth for all design decisions.

```sh
# Build the book
mdbook build      # outputs to book/
mdbook serve      # live preview at http://localhost:3000
```

## Repository Map

| Repo | Purpose |
|---|---|
| `playos-spec` | Specifications, ADRs, sprint plans (this repo) |
| `playos-platform-api` | Public `libplayos` C ABI |
| `playos-runtime` | Internal IPC and Wayland protocol |
| `playos-compositor` | wlroots Wayland compositor |
| `playos-shell` | Raylib controller-first UI and PlayOS Raylib backend |
| `playos-refdistro` | Buildroot OS image assembly |
| `playos-init` | PID 1 process supervisor |
| `playos-samples` | Sample games and reference applications |
| `playos-tools` | Host-side developer and OTA staging tools |
| `playos-foundation` | Shared foundation libraries and utilities |
| `playos-reference-devices` | Reference device configurations and images |
| `playos-cloud` | Cloud services (post-MVP) |
| `playos-marketplace` | Game store and marketplace (post-MVP) |

## Conventions

- **Language:** British/American English, present tense, imperative mood
- **Code blocks:** always specify language
- **ADRs:** append-only — supersede, never delete
- **Cross-links:** relative paths

See [`AGENTS.md`](AGENTS.md) for AI agent guidance.
