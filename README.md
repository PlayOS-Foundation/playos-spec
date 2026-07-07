# playos-spec

The official **PlayOS Platform Specification**, architecture, RFCs, and developer
documentation. It defines the platform contract independently of any single
implementation.

> **Motto:** Write Once. Play Anywhere. Own Everything.

## What is PlayOS?

PlayOS is an open, specification-first console platform. It defines a portable
platform model for building, running, packaging, distributing, and supporting
console-style games and applications across many devices, operating systems, and
hardware profiles.

PlayOS is **not** a single game engine, Linux distribution, launcher, or
marketplace. It is the **contract** that connects games, engines, runtimes,
devices, stores, and cloud services.

## Repository layout

```text
playos-spec/
├── book/            # The PlayOS Book (mdBook specification + guide)
│   ├── book.toml
│   └── src/
│       ├── SUMMARY.md
│       └── 00-front-matter/ … 99-appendices/
├── rfcs/            # Design proposals (RFC-0000 template + RFC-0001…0007)
├── adr/             # Accepted architecture decisions
├── schemas/         # Machine-readable formats (JSON Schema)
└── assets/          # Diagrams and branding
```

The relationship between them:

```text
book/    = official readable specification
rfcs/    = design proposals and discussions
adr/     = decisions that have been accepted
schemas/ = machine-readable formats used by tools and agents
```

## Building the book

The book is built with [mdBook](https://rust-lang.github.io/mdBook/).

```sh
# Install mdBook (requires Rust toolchain)
cargo install mdbook

# Serve locally with live reload
mdbook serve book --open

# Build static output into book/book/
mdbook build book
```

## Contributing

See [CONTRIBUTING.md](CONTRIBUTING.md) and the
[Governance and Process](book/src/16-governance-and-process/01-overview.md)
chapter. Changes that affect platform behavior should go through the
[RFC process](rfcs/0000-template.md) before implementation.

## License

See [LICENSE](LICENSE).
