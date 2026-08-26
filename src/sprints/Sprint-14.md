# Sprint 14 — Production Readiness

**Goal:** Deliver a signed preview release of PlayOS with a stable, versioned public Platform API, complete documentation, a validated release pipeline, and a full smoke-test pass on physical ROG Ally hardware.

**Primary Outcome:** PlayOS v0.3.0 is a signed, installable release that meets all 19 MVP criteria. The `libplayos` C ABI is documented and stable. A second developer can build a game using only the public API documentation.

**Prerequisites:** Sprint 13 complete — all MVP features implemented, Intel expansion validated.

---

## Why This Sprint Exists

All MVP features are implemented by the end of Sprint 13, but the project is not yet shippable. The public `libplayos` API has never been frozen, versioned, or documented, so any external developer would be building against a moving target. Release artifacts are produced by hand, there is no tag-triggered CI pipeline, recovery mode is only partially specified, and performance has never been measured. This sprint turns a working prototype into a signed, installable preview release: it freezes and documents the ABI, automates the release, proves the MVP criteria on real hardware, and closes the recovery and documentation gaps.

---

## Start Condition Checklist

- Sprint 13 complete: all MVP features are implemented and Intel expansion is validated.
- The eight public headers exist in `playos-platform-api/include/playos/`: `playos_audio.h`, `playos_display.h`, `playos_input.h`, `playos_lifecycle.h`, `playos_logging.h`, `playos_power.h`, `playos_storage.h`, `playos_system.h`.
- The public API is already versioned — `PLAYOS_API_VERSION 1` and `PLAYOS_API_VERSION_MAJOR/MINOR/PATCH 0.3.0` are set, with SONAME `libplayos.so.0` (S14-T1/T2 already done; this sprint formalizes and documents them).
- Release images are produced manually; there is no `release.yml` workflow.
- Recovery mode is partially specified: A/B rollback and factory reset exist from Sprint 10, but the recovery UI is not implemented.
- No performance baseline has been measured or documented.

---

## Decisions Locked for This Sprint

- **API version:** `PLAYOS_API_VERSION 1` (already set in `playos.h`).
- **Library version:** `PLAYOS_API_VERSION_MAJOR 0`, `MINOR 3`, `PATCH 0` (already set in `playos.h`).
- **SONAME:** `libplayos.so.0` for this release (already set in `CMakeLists.txt`).
- **Compatibility policy:** minor versions are backward-compatible; a major version bump is breaking.
- **Breaking-change process:** breaking changes after v0.3.0 require an RFC in `playos-spec`, an ADR, a major version bump, and a migration guide.
- **Release trigger and tag:** the pipeline runs on a version tag push such as `v0.3.0`.
- **Release artifacts:** `playos-v0.3.0-rog-ally-installer.img`, `playos-v0.3.0-rog-ally-update.playosb`, `playos-v0.3.0-sdk-headers.tar.gz`, plus SHA256 checksums and signatures.
- **Recovery rendering:** recovery must work without AMDGPU, using SimpleDRM or software rendering.
- **Performance targets:** the baseline table below is the acceptance target; any metric more than 2× over target is a documented gap.

---

## Scope

### In Scope

- Formal API compatibility review of all eight public headers.
- Versioning: `PLAYOS_API_VERSION 1`, library version, and SONAME `libplayos.so.0`.
- Doxygen documentation plus code examples and a getting-started guide for every API group.
- Game-developer guides in `playos-spec/docs/`.
- Tag-triggered release pipeline in `playos-refdistro/.github/workflows/release.yml`.
- Full 19-criterion MVP smoke test on physical ROG Ally hardware.
- Minimal recovery UI and recovery entry points.
- `playos-spec` completion: README, architecture, roadmap, platform API, IPC, security model, ADRs, docs.
- Performance baseline measurement and documentation.
- Production image hygiene: no debug tools, signed EFI artifact, signed update bundle.

### Explicitly Out of Scope

- Breaking changes to the public API (this sprint freezes v0.3.0; changes go through the post-v0.3.0 process).
- Production HSM-backed signing — the pipeline uses the development signing key only.
- Intel Vulkan (ANV) and multi-GPU work (deferred to future sprints).
- Network stack (no network in the MVP).

---

## Required Repository Changes

| Repo | Required work |
|---|---|
| `playos-platform-api` | API stability review, versioning and SONAME, Doxygen docs, code examples, getting-started guide |
| `playos-refdistro` | Release pipeline, recovery image/menu, performance measurement, production image hygiene |
| `playos-init` | Recovery entry logic: boot-count exceeded, button hold, repeated compositor failure |
| `playos-shell` | Minimal recovery UI (text or simple Raylib on SimpleDRM/framebuffer) |
| `playos-spec` | Authoritative reference completion: README, architecture, roadmap, platform API, IPC, security, ADRs, game-dev docs |
| All repos | Version tags and CHANGELOG updates for the `v0.3.0` release |

---

## Expected Files and Directories

### `playos-platform-api`

```text
include/playos/playos.h          # PLAYOS_API_VERSION 1 + PLAYOS_API_VERSION_* macros
CMakeLists.txt                    # SONAME libplayos.so.0
docs/api/                         # rendered Doxygen output
examples/                         # per-API-group code examples + minimal game
docs/getting-started.md           # minimal game using input, lifecycle, storage, logging
```

### `playos-refdistro`

```text
.github/workflows/release.yml     # tag-triggered v0.3.0 release pipeline
versions.lock                     # all component versions pinned
br2-external/board/ally/recovery/ # recovery menu sources and SimpleDRM/framebuffer config
br2-external/configs/playos_ally_recovery_defconfig
```

### `playos-init`

```text
src/recovery.c                    # recovery entry: boot-count, button hold, compositor-failure retry
```

### `playos-shell`

```text
src/recovery_menu.c               # minimal recovery menu: logs, factory reset, rollback, shutdown, reboot
```

### `playos-spec`

```text
README.md
architecture.md
roadmap.md
platform-api.md
runtime-ipc.md
security-model.md
adr/                              # all ADRs from Sprints 0-14
docs/                             # game-developer guides
```

---

## Agent Task Breakdown

### Task Status Grid

| Task ID | Task | Primary repo | Status | Notes / evidence |
|---|---|---|---|---|
| S14-T1 | Freeze the public API and set `PLAYOS_API_VERSION 1` | `playos-platform-api` | done | `playos.h` already defines `PLAYOS_API_VERSION 1` |
| S14-T2 | Set library version and SONAME `libplayos.so.0` | `playos-platform-api` | done | Already `0.3.0` + SONAME `libplayos.so.0` |
| S14-T3 | Complete Doxygen docs and code examples | `playos-platform-api` | not started | |
| S14-T4 | Implement the tag-triggered release pipeline | `playos-refdistro` | not started | |
| S14-T5 | Run the full 19-criterion MVP smoke test | `playos-refdistro` | not started | |
| S14-T6 | Implement recovery mode | `playos-init`, `playos-shell`, `playos-refdistro` | not started | |
| S14-T7 | Measure and document the performance baseline | `playos-refdistro` | not started | |
| S14-T8 | Complete `playos-spec` and game-developer guides | `playos-spec` | not started | |
| S14-T9 | Enforce production image hygiene and signed artifacts | `playos-refdistro` | in progress | Production defconfig, `production-build.yml`, and sign scripts exist; `release.yml` + SDK tarball pending |

Update the **Status** column as work progresses: `not started` → `in progress` → `blocked` or `done`.

### S14-T1 — Freeze the public API

Conduct a formal API compatibility review for every public header in `include/playos/`. For each API, check enum-value stability (adding is safe, removing is breaking), struct-layout stability (adding fields without versioning is ABI-breaking), function-signature stability, return-value semantics, error handling, and thread-safety guarantees. Set `PLAYOS_API_VERSION 1` in `playos.h`. Any breaking change found must be resolved before the freeze — either avoided or deferred behind the post-v0.3.0 process.

**Done when:** `playos.h` defines `PLAYOS_API_VERSION 1`, the review checklist is documented for all eight headers, and no unresolved breaking change remains.

### S14-T2 — Version the library and SONAME

Set `PLAYOS_API_VERSION_MAJOR 0`, `MINOR 3`, `PATCH 0`, and set the SONAME to `libplayos.so.0` in the `playos-platform-api` build. Both are already done; this task documents and verifies them. Document the compatibility policy: minor versions are backward-compatible; a major version bump is breaking. Document the breaking-change process: RFC in `playos-spec`, ADR, major version bump, and migration guide.

**Done when:** the built library reports SONAME `libplayos.so.0`, the version macros are exported, and the compatibility policy is documented in `playos-spec`.

### S14-T3 — Complete Doxygen documentation

Add Doxygen comments to every public symbol across the eight public headers, generate rendered docs into `docs/api/`, and write code examples for each API group. Write the "Getting Started" guide: create a minimal game that uses input, lifecycle, storage, and logging using only the public API.

**Done when:** Doxygen generates clean output with no undocumented public symbols, and the getting-started example compiles against the public headers.

### S14-T4 — Implement the release pipeline

Create `playos-refdistro/.github/workflows/release.yml`, triggered by a version tag push such as `v0.3.0`. The pipeline locks component versions in `versions.lock`, builds the production ROG Ally image (no debug tools, signed EFI artifact), builds the installer image, runs the QEMU boot test suite, verifies production lint, signs the EFI artifact and update bundle with the development key, and packages `installer.img`, `update.playosb`, and `sdk-headers.tar.gz` with SHA256 checksums and signatures before creating a GitHub Release.

**Done when:** pushing a test tag runs the pipeline end-to-end and produces all three artifacts plus checksums and a GitHub Release.

### S14-T5 — Run the full MVP smoke test

Run the complete 19-criterion MVP checklist on physical ROG Ally hardware and record the status of every criterion. The checklist covers boot from UEFI, PID 1, compositor ownership, shell persistence, wlroots/Mesa stack, Raylib shell rendering, public C ABI usage, lifecycle transport, supervised game launch, first-frame switching, hardware-accelerated rendering with controller input, system-button flow, resume, audio, clean exit and crash recovery, persistent saves, immutable system image, and graphics-free recovery.

**Done when:** a committed test report shows all 19 criteria passing on physical ROG Ally hardware.

### S14-T6 — Implement recovery mode

Implement a minimal recovery UI (text or simple Raylib on SimpleDRM/framebuffer). Recovery entry points are: boot count exceeds the A/B limit with both slots bad, a button hold at boot (e.g. Volume Down for 5 seconds), and `playos-init` entering recovery after repeated compositor failure. The menu offers: view system logs (`/data/log/`), factory reset (Sprint 10 logic), rollback to the previous system slot when available, shutdown, and reboot. Recovery must work without AMDGPU.

**Done when:** recovery is reachable from all three entry points and shows the menu with software/SimpleDRM rendering on hardware without AMDGPU.

### S14-T7 — Measure and document the performance baseline

Measure the performance targets on ROG Ally hardware: cold boot to shell < 5s, shell to game first frame < 3s, system button to overlay < 100ms, game exit to shell < 500ms, `sample-triangle` 60 FPS at native resolution, direct scanout confirmed in the compositor log, and idle shell CPU < 2%. File a performance issue in `playos-spec` for any target not met and document the gap.

**Done when:** a committed performance report contains measurements for every metric and lists any gaps with filed issues.

### S14-T8 — Complete `playos-spec`

Make `playos-spec` the authoritative reference: `README.md` (overview and navigation), `architecture.md`, `roadmap.md` (sprint plan and MVP criteria), `platform-api.md` (API contract and versioning policy), `runtime-ipc.md` (IPC protocol), `security-model.md`, `adr/` with all ADRs from Sprints 0–14, and `docs/` with the game-developer guides ("Building Your First PlayOS Game", lifecycle, storage, input, audio, and performance guides).

**Done when:** every listed spec document exists, is internally consistent, and links from `README.md`.

### S14-T9 — Enforce production image hygiene and signed artifacts

Confirm the production image ships without debug tools, the EFI artifact is signed with the development key, and the update bundle is signed. Verify the release pipeline's production-lint step fails on any debug binary. Produce the SDK headers tarball and confirm it compiles a minimal game on a Linux host.

**Done when:** the `v0.3.0` artifacts are signed and pass lint, and `sdk-headers.tar.gz` compiles a minimal game on a Linux host.

---

## Implementation Guidance

**Freeze first, document second.** Run the ABI review and set `PLAYOS_API_VERSION 1` before generating docs so the docs describe the frozen API, not a moving target.

**Treat the smoke test as the release gate.** The pipeline may pass in CI, but the release is not ready until the physical-hardware test report shows all 19 criteria passing.

**Recovery must be graphics-independent.** Do not make recovery depend on AMDGPU or hardware acceleration; validate it on SimpleDRM/software rendering.

**Keep production signing on the development key.** HSM-backed signing is post-MVP; do not introduce production key management in this sprint.

**Document performance gaps, don't silently relax targets.** If a metric misses, file an issue and record the measurement rather than editing the target.

**Do not break the ABI to fix docs.** If documentation reveals a design problem, defer it through the breaking-change process instead of changing the public headers this sprint.

---

## Verification and Evidence

| Evidence | How it is produced |
|---|---|
| Public API frozen and versioned | `playos.h` shows `PLAYOS_API_VERSION 1` and version macros |
| SONAME correct | `readelf -d libplayos.so` shows `SONAME libplayos.so.0` |
| API fully documented | Doxygen output with zero undocumented public symbols |
| Release pipeline works | CI log from a test tag produces installer, update bundle, and SDK tarball |
| Installer boots a clean device | Physical install of `playos-v0.3.0-rog-ally-installer.img` on a clean ROG Ally |
| Update applies via A/B | `update.playosb` applied through the A/B flow |
| MVP criteria met | Committed smoke-test report with all 19 criteria passing |
| SDK usable by a second developer | Minimal game compiled from `sdk-headers.tar.gz` on a Linux host |
| Recovery works | Boot into recovery from button hold and boot-count-exceeded; menu on SimpleDRM |
| Performance baseline | Committed performance report with measurements and filed gaps |
| Specs complete | `playos-spec` README/nav plus all referenced docs present and consistent |

---

## Acceptance Criteria

- [ ] All 19 MVP criteria pass on physical ROG Ally hardware
- [ ] `libplayos` public headers are fully documented (Doxygen)
- [ ] `PLAYOS_API_VERSION 1` defined; SONAME is `libplayos.so.0`
- [ ] "Building Your First PlayOS Game" guide is complete and tested
- [ ] Release pipeline produces signed `installer.img` and `update.playosb` from a tag push
- [ ] `playos-v0.3.0-rog-ally-installer.img` installs successfully on a clean ROG Ally
- [ ] `playos-v0.3.0-rog-ally-update.playosb` applies successfully via A/B update flow
- [ ] SDK headers tarball compiles a minimal game on a Linux host
- [ ] Recovery mode is reachable and shows the recovery menu without AMDGPU
- [ ] Performance baseline documented; no metric is more than 2× over target
- [ ] All ADRs from Sprints 0–14 are in `playos-spec/adr/`
- [ ] CI release pipeline passes end-to-end on a test tag
- [ ] `playos-spec` repository is complete and internally consistent

---

## Handoff to Sprint 15

Sprint 15 may assume:

- PlayOS v0.3.0 is a signed, installable release that passes all 19 MVP criteria.
- The public `libplayos` C ABI is frozen at `PLAYOS_API_VERSION 1`, versioned `0.3.0`, with SONAME `libplayos.so.0`.
- The SDK headers tarball exists and compiles a minimal game on a Linux host.
- Doxygen docs and game-developer guides are published in `playos-spec/docs/`.
- Recovery mode, performance baseline, and the tag-triggered release pipeline are in place.
- Breaking changes to the public API now require the RFC/ADR/major-bump/migration process.

---

## Exit Gate

PlayOS v0.3.0 is a signed, installable release that passes all 19 MVP criteria on physical ROG Ally hardware. The public API is documented, stable, and versioned. A second developer can build and run a game using only the published SDK.

*Previous: [Sprint 13](Sprint-13.md) | Next: [Sprint 15](Sprint-15.md)*
