# Platform SDK

The **PlayOS SDK** is the collection of libraries, tools, templates, and
samples a developer uses to build PlayOS applications. It is not a single
repository but a curated set of components that together form the developer
experience.

## What's in the SDK

| Component | Repository | What it gives the developer |
|---|---|---|
| Platform API | `playos-platform-api` | C++ headers + library (`PlayOS::Input`, `Capabilities`, `Lifecycle`) |
| Runtime launcher | `playos-runtime` | Launch games and wait for exit (`LaunchAndWait`) |
| Samples | `playos-samples` | `hello-playos` and future reference apps |
| Tools | `playos-tools` | Packaging (`gameos package`), templates, diagnostics (future) |

The SDK is the developer-facing layer. It must be usable on an SDK Target
(Windows, macOS, Linux desktop) without installing PlayOS as an operating
system. See the [Developer Guide](../15-developer-guide/01-overview.md).

## SDK vs Platform API

The **Platform API** is the contract (the `PlayOS::` headers). The **SDK** is
everything a developer needs to work with that contract: libraries, tools,
samples, templates, and documentation.

## Cross-platform

The SDK compiles on Windows, macOS, Linux, and supported SDK Targets (Android,
PS Vita). It does not require a PlayOS Runtime Device. A developer on Windows
can build, test, and even publish to the marketplace without owning PlayOS
hardware.

## Getting started

See the Developer Guide chapter
[Installing the SDK](../15-developer-guide/02-installing-the-sdk.md) and
[Your First PlayOS App](../15-developer-guide/03-your-first-playos-app.md).
