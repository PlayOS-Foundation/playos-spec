# Terminology

This chapter defines the canonical terminology used throughout the
PlayOS specification. All chapters, RFCs, ADRs, and schemas must use
these terms consistently.

## Core terms

| Term | Definition |
|---|---|
| **PlayOS Platform Specification** | The formal definition of the PlayOS platform (this book, schemas, and API headers). |
| **PlayOS Platform API** | The C++ API that applications use. Engine-agnostic, capability-based. |
| **PlayOS Runtime** | The software layer that provides process management, IPC, overlay, and cloud sync on Runtime Devices. |
| **PlayOS Shell** | The system UI: home screen, settings, overlay, launcher. |
| **PlayOS Cloud** | The hosted services: accounts, saves, leaderboards, marketplace. |
| **PlayOS Marketplace** | The application store. |

## Target types

| Term | Definition |
|---|---|
| **Runtime Device** | Hardware that runs the full PlayOS Runtime (e.g., ROG Ally, Orange Pi). |
| **SDK Target** | A development platform that runs PlayOS applications directly without the Runtime (e.g., Windows, macOS, Linux desktop). |

## Platform concepts

| Term | Definition |
|---|---|
| **Capability** | A named feature that a device may or may not support (e.g., `input.touch`). Queried at runtime. |
| **Device Profile** | A TOML file describing a device's hardware, input mappings, and capabilities. |
| **Platform Backend** | The OS-specific implementation of a Platform API module. |
| **Reference Implementation** | The canonical implementation maintained alongside the specification. |
| **Null Backend** | A conformant backend that reports all optional capabilities absent. |

## File formats

| Term | Definition |
|---|---|
| **Package (.gpk)** | A PlayOS application packaged for distribution. A ZIP file with a specific layout. |
| **Manifest** | The `manifest.json` inside a .gpk describing the application. |
| **Signature** | An Ed25519 detached signature verifying the package origin. |

## Avoided terms

Do not use these terms in the specification:

| Incorrect | Correct | Reason |
|---|---|---|
| "PlayOS OS" | "PlayOS Platform" | PlayOS is a platform, not an OS. |
| "Game engine" | "Rendering engine" or "Engine" | The Platform API is engine-agnostic. |
| "Platform check" | "Capability query" | Use capability queries, not platform checks. |
| "Raylib API" | "Platform API" | Raylib is the reference engine, not the API. |
| "Console" | "Runtime Device" | Use precise terminology. |

## Related

- [Glossary](06-glossary.md) — expanded definitions.
- [Capability Model](../05-capability-model/02-what-is-a-capability.md)
- [Normative Language](04-normative-language.md)
