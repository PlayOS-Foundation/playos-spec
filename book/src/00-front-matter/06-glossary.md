# Glossary

Expanded definitions of key terms. See [Terminology](05-terminology.md)
for the canonical short form.

## A

**ADR (Architecture Decision Record)**
: A document capturing an architectural decision, its context, and
  its consequences. Stored in `adr/`.

**API (Application Programming Interface)**
: See **PlayOS Platform API**.

**Application**
: An executable and its assets, packaged as a .gpk file, that runs
  on the PlayOS platform.

## B

**Backend**
: See **Platform Backend**.

## C

**Capability**
: A named feature that a device may or may not support. Capabilities
  are queried at runtime using `Capabilities::Has()`. Examples:
  `input.touch`, `power.battery`, `cloud.saves`.

**Capability Group**
: A namespaced collection of related capabilities (e.g., `input.*`,
  `display.*`, `storage.*`).

**Capability Registry**
: The canonical list of all capabilities maintained in
  `schemas/capability-registry.schema.json`.

**Certification**
: The process of verifying a device conforms to the PlayOS
  specification. Three levels: Platform API Compatible, Runtime
  Compatible, PlayOS Certified Hardware.

**Cloud API**
: The Protocol Buffer definitions for cloud services in
  `playos-cloud-proto`.

## D

**Device Profile**
: A TOML file describing a device's identity, input mappings,
  display properties, and capabilities. Schema:
  `schemas/device-profile.schema.json`.

**Delta Update**
: An update that downloads only the changed bytes between versions,
  not the full package.

## E

**Engine**
: A rendering engine (Raylib, SDL, Godot, custom). The Platform API
  is engine-agnostic.

**Entitlement**
: A record linking a player's account to an application they own.

## F

**Federation**
: The agreement between two store sources to share catalog data.

**Feature Level**
: A graded support level for an optional capability (e.g., "basic",
  "full").

## G

**.gpk (Package)**
: The PlayOS application package format. A ZIP file with a required
  layout including `manifest.json`, executables, and assets.

## I

**IPC (Inter-Process Communication)**
: The message-passing bus used by Runtime services to communicate.

## M

**Manifest**
: The `manifest.json` file inside a .gpk package. Describes the
  application's identity, permissions, capabilities, and metadata.

## N

**Null Backend**
: A conformant Platform Backend that reports all optional
  capabilities absent and returns safe defaults. Used as a fallback
  and porting starting point.

## O

**OEM (Original Equipment Manufacturer)**
: A company that manufactures hardware for PlayOS Runtime Devices.

**OEM Store**
: A marketplace operated by an OEM, customized for their devices.

**Overlay**
: System UI drawn on top of a running application (quick settings,
  notifications).

## P

**Platform API**
: See **PlayOS Platform API**.

**Platform Backend**
: The OS-specific implementation of a Platform API module (e.g.,
  `linux_input_backend.cpp`).

**PlayOS Cloud**
: The hosted cloud services (accounts, saves, leaderboards,
  marketplace).

**PlayOS Marketplace**
: The official application store.

**PlayOS Platform API**
: The C++ API that applications use. Defined in header files under
  `include/playos/`. Engine-agnostic and capability-based.

**PlayOS Platform Specification**
: The formal definition of PlayOS. This book, schemas, and API
  headers.

**PlayOS Runtime**
: The software layer on Runtime Devices that provides process
  management, IPC, overlay, cloud sync, and security sandboxing.

**PlayOS Shell**
: The system UI — home screen, settings, launcher, overlay.

## R

**Reference Device**
: Specific hardware maintained as a porting and certification
  baseline (e.g., ROG Ally, Orange Pi).

**Reference Implementation**
: The canonical implementation maintained alongside the
  specification. Used for conformance testing.

**RFC (Request for Comments)**
: A design proposal for a significant change to the specification.
  Stored in `rfcs/`.

**Runtime Device**
: Hardware that runs the full PlayOS Runtime. Provides the complete
  console experience.

## S

**SDK Target**
: A development platform (Windows, macOS, Linux) that runs PlayOS
  applications directly without the Runtime.

**Self-Hosting**
: The ability to run PlayOS Cloud services on your own hardware
  instead of using the hosted service.

## V

**Vendor Capability**
: A capability in the `vendor.*` namespace, specific to a particular
  hardware manufacturer. Must be certified.

## Related

- [Terminology](05-terminology.md) — canonical short definitions.
- [Normative Language](04-normative-language.md)
- [How to Read This Book](03-how-to-read-this-book.md)
