# Preface

PlayOS is an open console platform specification.

It defines a portable platform model for building, running, packaging, distributing, and supporting console-style games and applications across different devices, operating systems, and hardware profiles.

PlayOS is designed for developers who want the simplicity of lightweight game development, the consistency of a console platform, and the openness of modern cross-platform software.

The goal of PlayOS is not only to run games.

The goal is to define a platform contract that allows games, applications, runtimes, devices, stores, and cloud services to interoperate through a common specification.

---

## What PlayOS Is

PlayOS is a specification-first platform.

It defines:

- a Platform API,
- a runtime model,
- a capability system,
- a package format,
- a device profile model,
- a cloud and marketplace architecture,
- a certification model,
- and a governance process.

PlayOS may have many implementations.

There may be a Linux-based reference runtime, a Raylib-based reference shell, Android SDK targets, Windows development targets, PS Vita homebrew targets, ARM device targets, and future implementations that do not exist yet.

The specification is the contract that connects them.

---

## What PlayOS Is Not

PlayOS is not a game engine.

Game developers may use Raylib, SDL, Godot, a custom engine, or another technology. Raylib is the reference engine for samples, tutorials, and the reference shell, but the PlayOS Platform API must remain engine agnostic.

PlayOS is not only a Linux distribution.

A Linux-based runtime may be the first full reference implementation, but the platform is broader than one operating system.

PlayOS is not only a launcher.

The shell is one part of the system. The platform also includes APIs, packaging, capabilities, cloud services, marketplace integration, device profiles, certification, and developer tooling.

PlayOS is not only a marketplace.

The marketplace is a platform service. It must be possible to use PlayOS with official stores, community stores, OEM stores, private stores, or self-hosted infrastructure.

---

## Specification First

The PlayOS Book is the source of truth for the platform.

Implementation repositories must follow the specification rather than define platform behavior independently.

If a feature affects platform behavior, it should be specified here before it is implemented elsewhere.

This applies to:

- Platform API behavior,
- runtime behavior,
- package behavior,
- capability behavior,
- device profile behavior,
- certification behavior,
- cloud and marketplace behavior.

The purpose of this rule is to keep PlayOS coherent as it grows across multiple devices, engines, runtimes, and contributors.

---

## Runtime Devices and SDK Targets

PlayOS distinguishes between two major categories of targets.

A **Runtime Device** runs the full PlayOS environment. This may include the PlayOS Runtime, Shell, package execution layer, marketplace client, system services, compositor, and hardware integration.

Examples may include handheld PCs, Linux gaming devices, ARM boards, mini consoles, or future certified hardware.

An **SDK Target** supports the PlayOS Platform API, or a subset of it, without running the full PlayOS runtime.

Examples may include Windows, Linux desktop, macOS, Android, PS Vita homebrew, and other future targets.

This distinction allows PlayOS to support both full console-like devices and lightweight portable SDK integrations.

---

## Capability-Based Design

PlayOS applications should not rely on hardcoded platform checks.

Instead of asking:

```cpp
#ifdef ANDROID
```

applications should ask:

```cpp
if (PlayOS::Capabilities::Has(PlayOS::Capability::Touch)) {
    // Use touch input.
}
```

Capabilities allow PlayOS to support many devices without forcing every device to provide every feature.

A device may support touch but not cloud saves.

Another device may support cloud saves but not system overlay.

Another may support the Platform API but not the full runtime.

The capability model makes these differences explicit and discoverable.

---

## Engine Agnostic, Raylib Friendly

PlayOS is engine agnostic.

The Platform API must not depend on Raylib.

However, Raylib is an important reference technology for the project because it is simple, portable, open, and aligned with the goals of PlayOS.

The reference shell may be built with Raylib.

The first tutorials may use Raylib.

The official starter projects may use Raylib.

But the PlayOS Platform API must remain usable from other engines and frameworks.

The correct relationship is:

```text
Game or Application
    ├── Raylib / SDL / Godot / Custom Engine
    └── PlayOS Platform API
```

not:

```text
Game or Application
    └── Raylib-only PlayOS extension
```

---

## Self-Hostable by Design

PlayOS cloud and marketplace services should be self-hostable by design.

The platform may provide official hosted services, but the architecture should not require developers, communities, schools, OEMs, or private groups to depend on one central provider.

A healthy PlayOS ecosystem may include:

- official stores,
- community stores,
- OEM stores,
- private stores,
- local network stores,
- and fully self-hosted cloud services.

This principle protects openness and gives the platform long-term flexibility.

---

## Audience

This book is intended for:

- game developers,
- engine developers,
- runtime implementers,
- shell developers,
- cloud and marketplace developers,
- hardware porting teams,
- OEMs,
- contributors,
- certification reviewers,
- and AI coding agents working on PlayOS repositories.

Different readers may use different parts of the book.

Developers may begin with the Developer Guide.

Runtime implementers may begin with the Runtime Architecture.

Hardware teams may begin with the Device Model and Porting chapters.

Contributors should begin with the Platform Principles, RFC process, and governance model.

---

## Normative and Informative Content

Some parts of this book define required platform behavior.

Other parts explain design intent, examples, recommendations, or future direction.

The chapter on normative language explains how requirement words such as must, should, and may are used in this specification.

When in doubt, the Platform Specification, accepted RFCs, accepted ADRs, schemas, and conformance requirements take priority over examples or explanatory text.

---

## Status of This Book

This is an early version of the PlayOS Platform Specification.

The structure is expected to evolve.

APIs may change.

Package formats may change.

Capability names may change.

Runtime behavior may change.

However, the long-term goal is stability. Once the Platform API reaches a stable release, applications written against that version should continue to build and run on future compatible implementations whenever practical.

---

## The Platform Promise

PlayOS exists to make console-style software development simple, portable, and open.

A developer should be able to write a game, use the PlayOS Platform API, package it once, and target many environments through well-defined backends and capabilities.

A player should experience PlayOS as a fast, controller-first, console-like environment.

A hardware manufacturer should be able to adopt PlayOS without reinventing the entire software stack.

A community should be able to self-host the services it depends on.

This book defines the path toward that platform.

---

## Motto

**Write Once. Play Anywhere. Own Everything.**
