# Sprint 14 — Production Readiness

**Goal:** Deliver a signed preview release of PlayOS with a stable, versioned public Platform API, complete documentation, a validated release pipeline, and a full smoke test pass on physical ROG Ally hardware.

**Primary Outcome:** PlayOS v0.1.0 is a signed, installable release that meets all 19 MVP criteria. The `libplayos` C ABI is documented and stable. A second developer can build a game using only the public API documentation.

**Prerequisites:** Sprint 13 complete — all MVP features implemented, Intel expansion validated.

---

## Key Deliverables

### `libplayos` API Stability Review

Before the preview release, conduct a formal API compatibility review for every public header in `include/playos/`:

- `playos_input.h`
- `playos_lifecycle.h`
- `playos_system.h`
- `playos_storage.h`
- `playos_audio.h`
- `playos_power.h`
- `playos_logging.h`

**Review checklist for each API:**
- Are all enum values stable? (Adding values is safe; removing is breaking)
- Are all struct layouts stable? (Adding fields to a struct is ABI-breaking without versioning)
- Are all function signatures stable?
- Are all return value semantics documented?
- Is error handling documented?
- Are thread-safety guarantees documented?

**Versioning:**
- Set `PLAYOS_API_VERSION 1` in `playos.h`
- Set `LIBPLAYOS_VERSION_MAJOR 0`, `MINOR 1`, `PATCH 0`
- `libplayos.so.0` is the SONAME for this release
- Document the compatibility policy: minor versions are backward-compatible; major is breaking

**Breaking changes after v0.1.0 require:**
1. RFC in `playos-spec`
2. ADR documenting the decision
3. Major version bump
4. Migration guide

### Complete API Documentation

For each public header, generate or write:
- Doxygen comments on every public symbol
- A `docs/api/` directory in `playos-platform-api` with rendered docs
- Code examples for each API group
- A "Getting Started" guide: create a minimal game that uses input, lifecycle, storage, and logging

**Game developer documentation (in `playos-spec/docs/`):**
- "Building Your First PlayOS Game" — step-by-step guide
- "PlayOS Lifecycle Guide" — how FOREGROUND/BACKGROUND/TERMINATE affect a game
- "Storage and Save Data" — path conventions, atomic writes
- "Input Reference" — logical button and axis constants
- "Audio Guide" — Raylib audio integration with the PlayOS backend
- "Performance Guide" — power profiles, thermal limits, frame pacing

### Release Pipeline

Implement a production release pipeline in `playos-refdistro/.github/workflows/release.yml`:

**Triggered by:** Pushing a version tag (e.g., `v0.1.0`)

**Steps:**
1. Lock all component versions in `versions.lock`
2. Build production image for ROG Ally (no debug tools, signed EFI artifact)
3. Build installer image
4. Run QEMU boot test suite
5. Verify production lint (no debug binaries)
6. Sign the EFI artifact with the development signing key
7. Sign the update bundle
8. Package artifacts:
   - `playos-v0.1.0-rog-ally-installer.img`
   - `playos-v0.1.0-rog-ally-update.raucb`
   - `playos-v0.1.0-sdk-headers.tar.gz` (public headers + static lib)
   - SHA256 checksums and signatures
9. Create GitHub Release with artifacts and changelog

### Full MVP Smoke Test Pass

Run the complete MVP acceptance checklist on physical ROG Ally hardware and document results:

| # | MVP Criterion | Status |
|---:|---|---|
| 1 | ROG Ally boots from UEFI into PlayOS | |
| 2 | EFI artifact bootable | |
| 3 | `playos-init` as PID 1 | |
| 4 | Compositor owns DRM/KMS permanently | |
| 5 | `playos-shell` persistent and alive | |
| 6 | wlroots with AMDGPU, DRM/KMS, GBM, EGL, Mesa | |
| 7 | Shell renders via Raylib PlayOS backend | |
| 8 | Shell and game use public `playos-platform-api` C ABI | |
| 9 | Lifecycle transport internal to `playos-runtime` | |
| 10 | Shell requests launch; `playos-init` spawns and supervises | |
| 11 | First-frame switching | |
| 12 | Game renders with hw acceleration + controller input | |
| 13 | System button → PlayOS UI + game background | |
| 14 | Resume returns to same running game | |
| 15 | Game audio through ALSA | |
| 16 | Clean exit and crash → shell | |
| 17 | Games and saves persist on ext4 | |
| 18 | System image is immutable | |
| 19 | Recovery works without accelerated graphics | |

### Recovery Mode

Implement a minimal recovery UI (text or simple Raylib on SimpleDRM/framebuffer):

**Recovery entry points:**
- Boot count exceeds limit (A/B rollback) and both slots are bad
- User holds a button combination at boot (e.g., Volume Down for 5 seconds)
- `playos-init` explicitly enters recovery after repeated compositor failure

**Recovery menu options:**
- View system logs (read from `/data/log/`)
- Factory reset (calls the factory reset logic from Sprint 10)
- Rollback to previous system slot (if available)
- Shutdown
- Reboot

Recovery must work without AMDGPU — use SimpleDRM or software rendering.

### `playos-spec` Repository Completion

Ensure `playos-spec` is the authoritative reference:
- `README.md` — overview and navigation
- `architecture.md` — this document
- `roadmap.md` — sprint plan and MVP criteria
- `platform-api.md` — API contract and versioning policy
- `runtime-ipc.md` — IPC protocol specification
- `security-model.md` — security boundaries and hardening roadmap
- `adr/` — all Architecture Decision Records from all sprints
- `docs/` — all game developer guides

### Performance Baseline

Before shipping v0.1.0, measure and document the following on ROG Ally hardware:

| Metric | Target |
|---|---|
| Cold boot to shell | < 5 seconds |
| Shell to game first frame | < 3 seconds |
| System button to overlay visible | < 100ms |
| Game exit to shell visible | < 500ms |
| `sample-triangle` frame rate | 60 FPS at native resolution |
| Direct scanout activation | Confirmed via compositor log |
| Idle shell CPU usage | < 2% |

If any target is not met, file a performance issue in `playos-spec` and document the gap.

---

## Acceptance Criteria

- [ ] All 19 MVP criteria pass on physical ROG Ally hardware
- [ ] `libplayos` public headers are fully documented (Doxygen)
- [ ] `PLAYOS_API_VERSION 1` defined; SONAME is `libplayos.so.0`
- [ ] "Building Your First PlayOS Game" guide is complete and tested
- [ ] Release pipeline produces signed `installer.img` and `update.raucb` from a tag push
- [ ] `playos-v0.1.0-rog-ally-installer.img` installs successfully on a clean ROG Ally
- [ ] `playos-v0.1.0-rog-ally-update.raucb` applies successfully via A/B update flow
- [ ] SDK headers tarball compiles a minimal game on a Linux host
- [ ] Recovery mode is reachable and shows the recovery menu without AMDGPU
- [ ] Performance baseline documented; no metric is more than 2× over target
- [ ] All ADRs from Sprints 0–14 are in `playos-spec/adr/`
- [ ] CI release pipeline passes end-to-end on a test tag
- [ ] `playos-spec` repository is complete and internally consistent

---

## Repositories Primarily Involved

| Repo | Work |
|---|---|
| `playos-platform-api` | API stability review, Doxygen docs, versioning |
| `playos-spec` | All documentation, ADRs, game developer guides |
| `playos-refdistro` | Release pipeline, recovery mode, performance measurement |
| All repos | Version tags, CHANGELOG updates |

---

## Testing Approach

- Full MVP smoke test on physical ROG Ally (documented in a test report)
- Release pipeline dry run with a test tag before the real `v0.1.0` tag
- SDK test: compile and run `sample-triangle` using only the public SDK headers tarball on an Ubuntu host (cross-compilation)
- Recovery test: boot into recovery from both BIOS key hold and boot-count exceeded

---

## Exit Gate

PlayOS v0.1.0 is a signed, installable release that passes all 19 MVP criteria on physical ROG Ally hardware. The public API is documented, stable, and versioned. A second developer can build and run a game using only the published SDK.

*Previous: [Sprint 13](Sprint-13.md)*

---

## MVP Complete 🎮

With Sprint 14 complete, PlayOS has delivered its first meaningful release:

> A ROG Ally that boots directly from UEFI into a controller-first console experience, runs hardware-accelerated games with a stable public API, manages the full game lifecycle, and never shows the player a Linux prompt.
