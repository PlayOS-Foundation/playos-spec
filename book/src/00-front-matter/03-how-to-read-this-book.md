# How to Read This Book

The PlayOS Book is both a **specification** and a **guide**. Different
readers should read different parts.

## For application developers

You are building applications for PlayOS. Read:

1. [Platform Principles](../02-platform-principles/01-specification-first.md) —
   understand the philosophy.
2. [Capability Model](../05-capability-model/01-overview.md) —
   learn to query capabilities.
3. [Platform API](../06-platform-api/01-overview.md) — the API
   reference.
4. [Package Format](../10-package-format/01-overview.md) — how to
   package your application.
5. [Developer Guide](../15-developer-guide/01-overview.md) — getting
   started, tools, best practices.

## For platform implementors

You are building a PlayOS Runtime, Shell, or backend for a new OS or
device. Read:

1. [Platform Architecture](../03-platform-architecture/01-overview.md) —
   the big picture.
2. [Target Model](../04-target-model/01-overview.md) — what targets
   exist.
3. [Runtime Architecture](../08-runtime-architecture/01-overview.md) —
   how the Runtime works.
4. [Engine Integration](../07-engine-integration/01-overview.md) —
   integrating with rendering engines.
5. [Device Model and Porting](../12-device-model-and-porting/01-overview.md) —
   how to port to new hardware.

## For OEMs and hardware makers

You want to ship a PlayOS device. Read:

1. [Target Model](../04-target-model/01-overview.md) — Runtime Devices
   and SDK Targets.
2. [Certification](../14-certification/01-overview.md) — how to get
   certified.
3. [OEM Stores](../11-cloud-and-marketplace/18-oem-stores.md) — your
   own marketplace.
4. [Device Model and Porting](../12-device-model-and-porting/01-overview.md) —
   device profiles and backends.

## For contributors

You want to improve the specification. Read:

1. This chapter.
2. [Governance and Process](../16-governance-and-process/01-overview.md).
3. The [RFC template](../../rfcs/0000-template.md).
4. The [ADR template](../../adr/0000-template.md).

## Conventions

- **Bold** terms are defined in the [Glossary](06-glossary.md).
- Code blocks use C++ for Platform API examples, TOML for
  configuration, JSON for API responses.
- The [Normative Language](04-normative-language.md) chapter defines
  "must", "should", and "may."
- Capability names are written in `code font`: `input.touch`.
- Cross-references use relative links to other chapters.

## Reading order

The book is organized for sequential reading, but each chapter is
self-contained. Start wherever your needs take you.
