---
rfc: 0019
title: Update and Patch Distribution Model
status: Draft
authors: []
created: 2026-07-22
---

# RFC-0019: Update and Patch Distribution Model

> **Status:** Draft

## Summary

This RFC defines the full update lifecycle for PlayOS: how system and
application updates are discovered, downloaded, verified, and applied;
the release channel model; delta update strategy; rollback and
recovery; atomic update guarantees; and the CDN distribution
architecture.  It consolidates the [Updates and Patches][pkg-updates],
[Runtime Updates][rt-updates], and [Updates and CDN][cdn] chapters
into a single cross-cutting specification.

[pkg-updates]: ../book/src/10-package-format/13-updates-and-patches.md
[rt-updates]: ../book/src/08-runtime-architecture/15-updates.md
[cdn]: ../book/src/11-cloud-and-marketplace/16-updates-and-cdn.md

## Motivation

Updates are the most complex cross-cutting concern in PlayOS — they
touch the package format, the Runtime, the cloud, and the CDN.
Without a unified RFC:

- System updates and app updates would use different mechanisms.
- Delta updates (downloading only changed bytes) would be inconsistent.
- Rollback on failure would be undefined.
- The offline update model would be unspecified.
- Developers would not know what guarantees their update delivery has.

A unified update RFC ensures every layer of the stack handles updates
consistently.

## Guide-Level Explanation

PlayOS updates work like a modern console:

- **Apps update in the background.**  You're playing a game; an update
  downloads silently.  Next time you launch, it's the new version.
- **System updates are seamless.**  A/B partitioning means the update
  installs to the inactive slot while you're using the device.  Reboot,
  and you're on the new version.  If it fails, it rolls back
  automatically.
- **You choose the channel.**  Stable for everyone, beta for testers,
  nightly for the adventurous.
- **Save data is never touched.**  Updates replace code and assets,
  never your progress.

## Reference-Level Explanation

### 1. Update Lifecycle

```text
Check → Download → Verify → Notify → Install → Complete
  │        │         │         │         │
  │        │         │         │         └── Atomic swap
  │        │         │         └──────────── Player confirms (or auto)
  │        │         └────────────────────── Ed25519 signature check
  │        └──────────────────────────────── Background download
  └───────────────────────────────────────── Periodic poll or push
```

#### Step 1: Check

- **App updates:** Runtime polls each configured store source every
  6 hours (configurable).  Uses HTTP conditional requests
  (`If-None-Match`) to avoid re-downloading unchanged catalog data.
- **System updates:** Runtime polls the system update endpoint every
  24 hours.  Critical security patches trigger a push notification
  (when the device is online).
- **Manual check:** Player can check for updates at any time from
  Settings or the app's context menu.

#### Step 2: Download

- Downloads happen in the background with lowest I/O priority.
- Large downloads (> 100 MB) pause during game launch to avoid I/O
  contention — they resume after the game exits.
- Resumable downloads (HTTP Range requests) for interrupted transfers.
- Downloads are compressed (brotli for manifests, gzip for packages).

#### Step 3: Verify

- The downloaded `.gpk` signature is verified against the developer's
  public key (Ed25519, per RFC-0009).
- Verify-before-extract: the package is validated in a temp location
  before touching the installed version.
- Hash check: SHA-256 of the downloaded file matches the catalog
  entry.

#### Step 4: Notify

Notification behaviour depends on the player's update preference:

| Setting | Behaviour |
|---|---|
| **Auto-update** | Download and install without notification.  App updates apply on next launch. |
| **Notify** | Download silently.  Show non-blocking notification.  Player confirms install. |
| **Manual** | Show notification that update is available.  Player triggers download + install. |

#### Step 5: Install

- **App updates:** The Runtime extracts the new package to a staging
  directory.  On next app launch, the staging directory is atomically
  swapped with the current installation.  Save data and configuration
  (stored outside the package directory) are never touched.
- **System updates:** Applied to the inactive A/B slot.  The device
  reboots into the updated slot.  If boot fails (watchdog timer), the
  bootloader reverts to the previous slot.

#### Step 6: Complete

- Package database updated with new version.
- Old package files deleted (except when keeping one previous version
  for rollback).
- Update history logged.

### 2. Atomic Updates

All updates MUST be atomic — a failed update leaves the existing
installation intact.

#### App update atomiticy

```text
Before:  /apps/MyGame/v1.0.0/  (running)
         /data/MyGame/saves/   (untouched)

Update:  Download v1.1.0 → /tmp/update-abc123/
         Verify signature
         Extract to /apps/MyGame/v1.1.0.staging/
         
Install: mv /apps/MyGame/v1.1.0.staging /apps/MyGame/v1.1.0  (atomic rename)
         Update symlink: /apps/MyGame/current → v1.1.0

After:   /apps/MyGame/v1.0.0/  (kept for rollback)
         /apps/MyGame/v1.1.0/  (new version)
         /apps/MyGame/current → v1.1.0
         /data/MyGame/saves/  (untouched)
```

#### System update atomiticity (A/B)

```text
Slot A: v1.0.0 (active, running)
Slot B: v1.0.0 (inactive)

Update: Download v1.1.0 → write to Slot B
         Verify signature
         Set Slot B as "try-boot" with watchdog timer (60s)

Reboot:  Bootloader tries Slot B
         If boot succeeds → Slot B marked as active
         If boot fails (watchdog expires) → bootloader reverts to Slot A

After:   Slot A: v1.0.0 (inactive, kept for rollback)
         Slot B: v1.1.0 (active)
```

### 3. Release Channels

| Channel | Audience | Update Frequency | Stability |
|---|---|---|---|
| `stable` | All players | Per MINOR release | Production-tested |
| `beta` | Opt-in testers | Per PATCH release | Pre-release testing |
| `nightly` | Developers | Every commit | May be unstable |

#### Channel selection

Per-application channel selection in the Shell:

```
┌──────────────────────────────────────────┐
│  MyGame — Release Channel                │
├──────────────────────────────────────────┤
│  ○ Stable (v1.0.0)                       │
│  ● Beta (v1.1.0-beta.3)                  │
│  ○ Nightly (v1.1.0-nightly.20260722)     │
│                                          │
│  Switching channels will download the     │
│  latest version in the selected channel. │
│  Save data is preserved.                 │
│                                          │
│  [Switch to Beta]                        │
└──────────────────────────────────────────┘
```

#### Version comparison across channels

Versions are compared numerically regardless of channel:
`1.1.0 > 1.0.0`, even if `1.1.0` is in `nightly` and `1.0.0` is in
`stable`.  The Runtime always offers the highest version in the
selected channel.

#### Channel migration

- Switching from `beta` to `stable`: if the stable version is lower,
  the player is downgraded.  Save data backward compatibility is the
  developer's responsibility (per RFC-0008).
- Switching from `stable` to `nightly`: downloads the latest nightly.
  Downgrading back to `stable` uses the same atomic swap mechanism.

### 4. Delta Updates

Delta updates download only the bytes that changed between versions.
This is critical for large games with frequent updates.

#### Delta computation

Deltas are computed server-side using **bsdiff**:

```text
v1.0.0.gpk (1 GB)  +  v1.1.0.gpk (1 GB)
    ↓
bsdiff(v1.0.0.gpk, v1.1.0.gpk) → v1.0.0-to-v1.1.0.patch (50 MB)
```

The patch is cached alongside the full package on the CDN.

#### Delta application

The device downloads the patch and applies it locally:

```text
v1.0.0.gpk (on device) + v1.0.0-to-v1.1.0.patch (downloaded)
    ↓
bspatch(v1.0.0.gpk, patch) → v1.1.0.gpk
    ↓
Verify Ed25519 signature of v1.1.0.gpk
```

If patch application fails, the device falls back to downloading the
full `.gpk`.

#### Delta update strategy

| Scenario | Strategy |
|---|---|
| Update from previous version | Download delta patch |
| Update from 2+ versions behind | Download delta patch for each intermediate version, OR full download if total patch size > 50% of full |
| Fresh install | Full download |
| Delta application fails | Fall back to full download |

#### Requirements

- Delta patches MUST produce a byte-identical result to the full
  download (verified by SHA-256 hash).
- The patched result MUST be signature-verified before installation.
- The Runtime MUST measure available storage before downloading deltas
  (patching requires temporary space for both old and new versions).

### 5. Rollback

#### App rollback

The Runtime keeps the **previous version** of each app for rollback:

```
/apps/MyGame/
├── v1.0.0/          # Previous version (kept for rollback)
├── v1.1.0/          # Current version
└── current → v1.1.0
```

If the player reports issues, they can roll back from the app's
context menu:

```
┌──────────────────────────────────────────┐
│  MyGame                                  │
│  Version 1.1.0                           │
│                                          │
│  [Launch]                                │
│  [Check for Updates]                     │
│  [Roll Back to v1.0.0]                   │
│  [Uninstall]                             │
└──────────────────────────────────────────┘
```

Rollback preserves save data.  Only one previous version is kept;
rolling back again requires reinstalling from the store.

#### System rollback

A/B partitioning provides automatic rollback on boot failure.  Manual
rollback (Settings → System → Recovery) is also available.

### 6. CDN Architecture

```
Developer uploads .gpk
    ↓
Package Storage (S3/MinIO)
    ↓
CDN Origin
    ↓
Edge Nodes (worldwide)
    ↓
Devices (download from nearest edge)
```

#### CDN requirements

| Requirement | Mechanism |
|---|---|
| Global distribution | Multiple edge locations |
| Low latency | Geographic routing (DNS-based) |
| High availability | Redundant edge nodes |
| Resumable downloads | HTTP Range requests |
| Content addressing | SHA-256 hash in URL path |
| Cache invalidation | New version = new hash = new cache entry |

#### Reference deployment

The reference deployment uses **Cloudflare R2 + Cloudflare CDN**.
Self-hosted deployments can use:

| Scale | Solution |
|---|---|
| Personal | MinIO + nginx (single server) |
| Community | MinIO + multi-region replicas |
| Enterprise | S3 + CloudFront, or MinIO + custom CDN |

### 7. Security Requirements

- All updates MUST be delivered over TLS 1.3.
- All packages MUST be Ed25519 signature-verified before installation.
- System updates MUST be signed with the platform signing key
  (separate from developer keys).
- The platform MUST NOT be downgradeable to a version with known
  security vulnerabilities (minimum version check in the update
  manifest).
- Update metadata (version list, channel info) MUST be signed to
  prevent tampering.

### 8. Offline Updates

- **App updates:** Require network.  No offline update mechanism.
- **System updates:** Can be applied from USB storage (Settings →
  System → Update from USB).
- **Update notification cache:** If a device is offline during a poll,
  it retries on the next poll interval.  Push notifications for
  critical patches wait until the device reconnects.

## Drawbacks

- A/B partitioning for system updates halves available storage for the
  OS partition.
- Delta updates add server-side computation cost (bsdiff for every
  version pair) and increase package storage requirements.
- Keeping previous app versions for rollback doubles storage per app.
- The 6-hour poll interval means critical security patches could be
  delayed up to 6 hours (mitigated by push notifications).

## Rationale and Alternatives

- **Full download only (no deltas):** Simpler but wasteful — a 1 GB
  game with a 1 MB code change forces a 1 GB download.  Delta updates
  are essential for mobile/handheld devices with metered connections.
- **Single-slot updates (no A/B):** Saves storage but risks bricking
  on failed update.  A/B is the console standard (Nintendo Switch,
  Steam Deck) for reliability.
- **Streaming updates (download while playing):** Would reduce wait
  time but introduces I/O contention and complexity.  Background
  download with install-on-relaunch is simpler and more predictable.
- **No rollback:** Simpler but punishes players for updating.  Keeping
  one previous version provides a safety net without excessive storage
  cost.

## Prior Art

- **Steam / SteamOS updates** (delta patching, background downloads,
  A/B system updates): the closest analogue.  PlayOS follows a very
  similar model.
- **Nintendo Switch system updates** (A/B partitioning, reboot to
  apply): reference for console-grade system update reliability.
- **Android A/B (seamless) updates:** Google's A/B update mechanism
  for Android devices.  PlayOS adopts the same slot-based model.
- **Courgette / bsdiff:** Google's Courgette and Colin Percival's
  bsdiff are the standard binary diff tools.  PlayOS uses bsdiff for
  its simplicity and proven track record.
- **ChromeOS updates** (A/B, verified boot, delta updates): reference
  for the full update chain from CDN to verified boot.

## Unresolved Questions

- What is the maximum delta patch computation window?  Computing deltas
  for every version pair (N²) becomes expensive at scale.  A sliding
  window (last 5 versions) is proposed.
- How are system update signing keys managed?  The platform signing key
  must be stored in a hardware security module (HSM) and never
  exposed to the build pipeline.
- Should players be able to skip updates?  System updates cannot be
  skipped indefinitely (security).  App updates can be skipped, but
  multiplayer may require the latest version.
- How are very large games (100+ GB) handled?  Delta updates become
  more important; streaming installs (play while downloading) may be
  needed.

## Future Possibilities

- **Streaming install:** Play a game while it downloads — the first
  playable chunk (menus + first level) is prioritized.
- **Peer-to-peer update distribution:** Devices on the same LAN share
  update data to reduce CDN bandwidth.
- **Differential system updates:** Apply only changed files to the OS
  partition instead of full-slot updates, reducing write amplification
  on flash storage.
- **Update staging:** Developers can stage an update (upload but don't
  publish) and schedule a release date/time.
- **Update analytics:** Track update adoption rates, download
  completion rates, and rollback rates per version.
