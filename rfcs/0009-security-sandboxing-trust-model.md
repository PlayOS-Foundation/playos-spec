---
rfc: 0009
title: Security, Sandboxing, and Trust Model
status: Draft
authors: []
created: 2026-07-22
---

# RFC-0009: Security, Sandboxing, and Trust Model

> **Status:** Draft

## Summary

This RFC defines the PlayOS security architecture: the layered trust
model, application sandboxing, permission enforcement, code signing PKI,
secure boot and update chain, runtime service authentication, privacy
guarantees, and parental controls.  It formalises the security pillars
described in the [Security and Trust principle][sec-principle] and the
[Runtime Security Model][runtime-sec] chapter, and provides the
foundation for the Part XIII (Security, Privacy and Trust) book chapters.

[sec-principle]: ../book/src/02-platform-principles/09-security-and-trust.md
[runtime-sec]: ../book/src/08-runtime-architecture/17-security-model.md

## Motivation

The current specification describes security in fragments — the runtime
security model chapter, the permissions and signing chapters in the
package format part, and stub chapters for an entire Part XIII.  Without
a unified security RFC, there is no single source of truth for:

- How trust flows from hardware to application.
- What isolation guarantees the platform provides.
- How permissions are declared, granted, enforced, and revoked.
- How code signing keys are managed and revoked.
- What privacy guarantees players can rely on.

A unified RFC lets implementers build a coherent security model, lets
certification define security requirements, and lets developers
understand the guarantees their applications run under.

## Guide-Level Explanation

PlayOS security is built on four layers of trust, each enforced by the
layer below it:

```text
Player trusts the device   ← hardware root of trust, verified boot
    ↓
Device trusts the Runtime  ← signed OS image, read-only root
    ↓
Runtime trusts the package ← Ed25519 signature verification
    ↓
Application gets permissions ← player consent, runtime enforcement
```

**For players:** Every app runs in a sandbox.  Apps can't read each
other's data.  Before an app can use the microphone, internet, or your
save files, you must approve it.  You can change your mind later in
Settings.

**For developers:** You declare what your app needs in `manifest.json`.
The runtime enforces it — if you didn't declare `network.internet`, your
socket calls fail.  You don't need to implement sandboxing; the platform
does it.

**For hardware vendors:** Certified devices must implement verified boot
and a hardware root of trust.  The reference implementation uses Linux
namespaces, seccomp, and cgroups; equivalent mechanisms on other kernels
are acceptable.

## Reference-Level Explanation

### 1. Trust Model

Trust flows in one direction only.  No layer can bypass the layer above.

| Layer | Establishes Trust Via | Enforced By |
|---|---|---|
| Hardware → OS | Verified boot (signed kernel + initramfs) | UEFI Secure Boot or equivalent |
| OS → Runtime | Signed OS image, read-only root filesystem | dm-verity / fs-verity |
| Runtime → Package | Ed25519 detached signature | Runtime Security Monitor |
| Package → Permissions | Player consent at install time | Permissions layer in Platform API |

The runtime MUST NOT run unsigned packages outside developer mode.  The
OS MUST NOT mount a writable root filesystem in production.  The
bootloader MUST NOT load an unsigned kernel.

### 2. Sandboxing

Every application runs in an isolated sandbox.  The sandbox MUST
guarantee:

1. **Filesystem isolation** — an application cannot read another
   application's save data, configuration, or package files.
2. **Network isolation** — an application cannot listen on privileged
   ports or inspect another application's traffic.
3. **Process isolation** — an application cannot signal, ptrace, or
   otherwise interfere with another application's process.
4. **Resource limits** — an application cannot exhaust system memory,
   CPU, or disk I/O to deny service to other applications.

#### Linux reference implementation

| Mechanism | Purpose |
|---|---|
| User namespaces | Each app has its own UID/GID; no shared user |
| Mount namespaces | Restricted filesystem view; only app bundle + save dir |
| PID namespaces | App sees only its own process tree |
| Network namespaces | Optional; app may be denied network entirely |
| Seccomp (BPF) | Syscall filtering; deny `ptrace`, `mount`, `reboot`, etc. |
| Cgroups v2 | Memory, CPU, and I/O limits per application |

#### Windows reference implementation

| Mechanism | Purpose |
|---|---|
| Job objects | Process group management, resource limits |
| Restricted tokens | Drop privileges; remove SeDebugPrivilege, etc. |
| AppContainer | Per-app SID, filesystem and network isolation |
| Integrity levels | Low integrity for app processes |

#### SDK Target behaviour

On SDK Targets (Windows, macOS, Android) where full sandboxing may not
be available, the Platform API MUST still enforce filesystem isolation
(separate save directories per app ID) and permission checks.  SDK
Targets are not required to provide kernel-level isolation.

### 3. Permission Architecture

#### Permission lifecycle

```text
Declare (manifest.json)
    → Present (install-time UI)
        → Consent (player grants/denies)
            → Enforce (Platform API checks)
                → Revoke (player changes in Settings → immediate)
```

#### Declaration

Permissions are declared in `manifest.json`:

```json
{
  "permissions": ["storage.local", "network.internet"]
}
```

Undeclared permissions MUST NOT be grantable at runtime.

#### Player consent

At install time, the Shell presents the permission list to the player.
Permissions are grouped by risk level (low, medium, high).  The player
may grant all, deny specific permissions, or deny all.  If the player
denies a permission required for core functionality, the app is
installed but the player is informed that it may not function correctly.

#### Enforcement

The Platform API checks permissions before every protected operation:

```cpp
// Internally (not visible to app developer):
if (!Permissions::GetState("network.internet") == PermissionState::Granted) {
    return Error::PermissionDenied;
}
```

The runtime MUST NOT grant permissions silently.  If a permission is
revoked while an app is running, in-flight operations MAY complete but
new requests MUST be denied.

#### Revocation

The player may revoke any permission at any time through Settings.
Revocation takes effect immediately.  The runtime sends a
`PermissionsChanged` event to the running application.

### 4. Code Signing PKI

#### Key format

All signatures use **Ed25519**.  The signature is a detached file at the
package root:

```text
MyGame.gpk
├── manifest.json
├── bin/
├── assets/
└── signature           ← Ed25519 detached signature
```

#### Signing flow

1. Developer generates an Ed25519 key pair.
2. Developer registers the public key with a marketplace or includes it
   in a self-distributed package.
3. The package archive (excluding `signature`) is hashed with BLAKE2b.
4. The hash is signed with the developer's private key.
5. The signature is placed at `signature` in the package root.

#### Verification flow

1. Runtime reads the `signature` file.
2. Runtime computes BLAKE2b hash of the package contents.
3. Runtime verifies the Ed25519 signature against the developer's public
   key (obtained from marketplace key registry or embedded certificate).
4. If verification fails or no signature is present (outside developer
   mode), installation is rejected.

#### Key management

- Developers manage their own private keys.  Marketplaces MUST NOT hold
  developer private keys without explicit opt-in.
- Key compromise: the developer revokes the old public key via the
  marketplace API.  Packages signed with the revoked key are rejected.
- Already-installed packages with a revoked key continue to run but
  cannot be updated.
- The marketplace MAY offer a managed signing service where it holds the
  key and signs on the developer's behalf.

### 5. Secure Boot and Update Chain

#### Verified boot (reference Linux)

```text
UEFI Secure Boot
    → signed shim
        → signed GRUB
            → signed Linux kernel + initramfs
                → dm-verity root filesystem (read-only)
                    → PlayOS Runtime
```

The root filesystem MUST be read-only in production.  Writable state
(save data, configuration, package installs) lives on a separate
partition.

#### Secure updates

- System updates are delivered as signed deltas over TLS 1.3.
- The update payload is verified against a platform-signing key before
  application.
- The platform MUST NOT be downgradeable to a vulnerable version without
  explicit user action (e.g., factory reset).
- A/B partitioning is RECOMMENDED: the update is applied to the inactive
  slot, verified, then activated on next boot.  If boot fails, the
  bootloader reverts to the known-good slot.

### 6. Runtime Service Authentication

Runtime services (IPC bus, Security Monitor, Cloud Sync, etc.)
authenticate to each other using code signing.  The Security Monitor
verifies each service's identity before allowing it to join the IPC bus.

- Services are signed with the same Ed25519 key infrastructure as
  packages.
- The Security Monitor maintains a service identity registry.
- A compromised application MUST NOT be able to impersonate a runtime
  service — the IPC bus enforces sender identity at the transport layer.

### 7. Privacy Guarantees

PlayOS makes the following privacy guarantees:

| Guarantee | Mechanism |
|---|---|
| No telemetry without consent | Opt-in only; disabled by default |
| Crash reports are anonymized | Stripped of PII before upload |
| Cloud saves are encrypted at rest | AES-256-GCM, per-user keys |
| No advertising ID | No persistent device or user identifier for tracking |
| Local-first by default | All features work without cloud; cloud is additive |
| Data portability | Players can export all save data and configuration |

### 8. Parental Controls

Parental controls are a first-class platform feature:

| Control | Mechanism |
|---|---|
| Content rating restrictions | ESRB/PEGI rating filter; apps above threshold require PIN |
| Play time limits | Daily and session time limits; enforced by Lifecycle Manager |
| Purchase restrictions | PIN required for purchases; spending limits |
| Communication controls | Disable multiplayer, chat, friend requests |
| Web filtering | DNS-based content filtering (optional) |

Parental controls are configured through Settings and protected by a
device PIN.  They apply to all user accounts on the device.

### 9. Supply Chain Security

- All packages in the marketplace catalog MUST be signed.
- The marketplace MUST record the source repository and build provenance
  for each package version.
- Reproducible builds are RECOMMENDED: two builds from the same source
  SHOULD produce byte-identical packages.
- Dependency manifests (for apps that bundle libraries) SHOULD include
  SBOM (Software Bill of Materials) in SPDX format.

## Drawbacks

- Full sandboxing (user namespaces, seccomp, cgroups) requires a
  relatively modern Linux kernel (5.10+) and may not be available on all
  target devices.  SDK Targets cannot provide equivalent isolation.
- Per-application sandboxes increase memory overhead (kernel structures
  per namespace).
- Ed25519 key management places a burden on developers who are not
  familiar with cryptographic key hygiene.
- Opt-in telemetry reduces the data available for platform improvement
  compared to opt-out models used by most commercial consoles.

## Rationale and Alternatives

- **Linux namespaces vs. pure userspace sandboxing (e.g., Flatpak):**
  Namespaces provide kernel-enforced isolation that cannot be bypassed by
  a compromised userspace process.  Pure userspace sandboxing (ptrace,
  LD_PRELOAD interception) is bypassable.  Kernel mechanisms are chosen.
- **Ed25519 vs. RSA/ECDSA:** Ed25519 is faster, produces smaller
  signatures, and has fewer implementation pitfalls than ECDSA.  It is
  the modern standard for code signing (used by Signify, Minisign, age).
- **All-or-nothing permissions vs. runtime prompts:** Runtime prompts
  (Android/iOS model) create prompt fatigue.  PlayOS uses install-time
  consent with post-install revocation, which is the console standard
  (Nintendo Switch, PlayStation).
- **Opt-in vs. opt-out telemetry:** Consistent with the Privacy by
  Default principle and GDPR requirements.  Opt-in ensures the platform
  is trustworthy in privacy-sensitive markets (EU, education, enterprise).
- **A/B updates vs. single-slot:** A/B updates provide a fallback if an
  update fails, critical for a console that must always boot.  Single-slot
  updates risk bricking.  A/B is RECOMMENDED.

## Prior Art

- **Android security model** (sandboxing via UID separation, permissions,
  signed APKs): the closest analogue.  PlayOS follows a similar model but
  with install-time consent instead of runtime prompts, and console-grade
  isolation.
- **Nintendo Switch** (horizon OS): microkernel isolation, signed titles,
  no runtime permission prompts.  PlayOS aims for similar security
  properties on a Linux foundation.
- **Flatpak** (Bubblewrap, portals, static permissions): demonstrates
  that Linux namespace-based sandboxing is practical for graphical
  applications.  PlayOS adopts the namespace approach but with its own
  permission model.
- **ChromeOS** (verified boot, A/B updates, read-only root): reference
  for the secure boot and update chain.  PlayOS adopts the A/B update
  model and dm-verity pattern.
- **Signify / Minisign** (Ed25519 detached signatures): reference for the
  signing model.  PlayOS uses the same cryptographic primitives.

## Unresolved Questions

- What is the minimum Linux kernel version for the reference
  implementation? 5.10 (LTS) is proposed but needs validation on target
  devices.
- Should the parental controls PIN be per-device or per-account?
  Per-device is simpler for shared family devices.
- How are marketplace signing keys distributed to devices that use
  third-party private stores?  Federation protocol needs to address key
  distribution.
- What is the certification requirement for the hardware root of trust?
  TPM 2.0 vs. firmware-based?

## Future Possibilities

- **Hardware-backed keystore:** Use TPM or secure enclave for storing
  platform signing keys and per-device secrets.
- **Integrity attestation:** Remote attestation that a device is running
  an unmodified PlayOS image, enabling anti-cheat without kernel
  anti-cheat modules.
- **Per-permission runtime prompts:** As an alternative to all-or-nothing
  install-time consent, allow the player to grant individual permissions
  at first use (with a "don't ask again" option).
- **Network sandboxing per-domain:** Beyond binary network access,
  restrict network access to specific domains declared in the manifest.
- **Multi-user device support:** Multiple player profiles on one device,
  each with their own save data, permission grants, and parental controls.
