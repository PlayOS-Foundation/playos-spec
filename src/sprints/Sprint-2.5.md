# Sprint 2.5 — Cross-Sprint Audit Remediation

**Goal:** Address 7 actionable findings from the comprehensive Sprint 0–2 audit (2026-08-08) — eliminate code duplication, harden reproducibility, fix structural drift, and clean up deprecated artifacts before Sprint 3 hardware bring-up begins.

**Primary Outcome:** All 7 audit findings resolved. `make setup` produces a bit-identical source tree from `versions.lock`. IPC code lives in one canonical location. Board files match spec layout. Deprecated files removed. The foundation is clean before physical hardware work starts.

**Status:** 🔴 Not started

**Prerequisites:** Sprint 2 implementation complete. Audit report produced (session checkpoint `004-sprint-1-2-implementation-audi.md`).

---

## Why This Sprint Exists

The Sprint 0–2 audit found no critical bugs, but identified 7 medium-risk issues that, if left unaddressed, will compound:

1. **IPC duplication** — two diverging copies of server/client code will create subtle bugs as both evolve.
2. **Version pinning is decorative** — `versions.lock` has precise SHAs but `make setup` ignores them, making builds non-reproducible.
3. **Structural drift** — file locations don't match the spec, confusing new contributors.
4. **Dead code** — deprecated files still on disk.
5. **Stub drift** — `playos-runtime` Buildroot package hasn't progressed since Sprint 0 despite the IPC library existing.

Sprint 3 touches physical hardware, kernel configs, firmware, and a new public API header. Cleaning up these issues now prevents Sprint 3 from inheriting a messy foundation.

---

## Start Condition Checklist

- [ ] Sprint 2 QEMU build still works (or at minimum the code compiles).
- [ ] Audit report has been reviewed and the 7 findings are understood.
- [ ] All 6 repos are accessible and writable.

---

## Decisions Locked for This Sprint

- **Canonical IPC home:** `playos-refdistro/src/playos-init/ipc/` — the IPC code lives where it's consumed (PID 1). The `playos-runtime` copy is **removed** and `playos-runtime` depends on playos-init's IPC source via a shared include path or becomes a protocol-only package.
- **Version pinning enforcement:** `make setup` reads `versions.lock` and checks out the exact SHA for every component.
- **Board directory location:** remains under `br2-external/board/` (correct for Buildroot `BR2_EXTERNAL` paths). The Sprint-0.md spec is updated to reflect reality.
- **Restart policy:** 3 restarts / 60 seconds is the correct value. Sprint-1.md is updated to match.
- **No new features.** This sprint is pure remediation.

---

## Scope

### In Scope

- Unify IPC code into one canonical location
- Make `make setup` checkout pinned SHAs from `versions.lock`
- Update Sprint-0.md and Sprint-1.md to reflect actual file locations and restart policy
- Remove deprecated `linux.fragment`
- Fill GPT partition GUID search TODO in `mount.c`
- Update `playos-runtime` Buildroot package to install protocol XML into staging
- Verify the QEMU Buildroot build still passes after all changes
- Clean any build artifacts left in repos

### Explicitly Out of Scope

- New features or protocol changes
- Sprint 3 hardware work
- Shell, overlay, or game launch implementation
- CI pipeline redesign
- playos-platform-api or playos-shell implementation

---

## Required Repository Changes

| Repo | Required work |
|---|---|
| `playos-refdistro` | IPC unification, Makefile version pinning, mount.c GPT GUID, remove linux.fragment, update playos-runtime package, update board paths in docs |
| `playos-runtime` | Remove duplicated IPC source files, keep only protocol XML + headers, update CMakeLists.txt |
| `playos-spec` | Update Sprint-0.md (board location), Sprint-1.md (restart policy), Sprint-2.md (acceptance criteria status) |

---

## Agent Task Breakdown

Every task below is independently checkable.

### Task Status Grid

| Task ID | Task | Primary repo | Status | Notes / evidence |
|---|---|---|---|---|
| S2.5-T1 | Unify IPC code — make playos-init/ipc/ canonical, remove playos-runtime duplicate | `playos-refdistro`, `playos-runtime` | not started | |
| S2.5-T2 | Enforce version pinning in `make setup` | `playos-refdistro` | not started | |
| S2.5-T3 | Update Sprint-0.md: board directory location | `playos-spec` | not started | |
| S2.5-T4 | Update Sprint-1.md: restart policy (3/60s) | `playos-spec` | not started | |
| S2.5-T5 | Remove deprecated `linux.fragment` | `playos-refdistro` | not started | |
| S2.5-T6 | Implement GPT partition GUID search in mount.c | `playos-refdistro` | not started | |
| S2.5-T7 | Wire playos-runtime Buildroot package to install protocol XML | `playos-refdistro` | not started | |
| S2.5-T8 | Update Sprint-2.md acceptance criteria after QEMU verification | `playos-spec` | not started | |

---

### S2.5-T1 — Unify IPC Code Into One Canonical Location

**Finding:** IPC source files (`ipc_framing.c`, `ipc_server.c`, `ipc_client.c`, `lifecycle_fd.c`, `ipc.h`) exist in both `playos-runtime/` (builds as `libplayos-ipc.a`) and `playos-refdistro/src/playos-init/ipc/` (compiled directly into init). The header and framing are identical, but server/client implementations have diverged.

**Decision:** `playos-refdistro/src/playos-init/ipc/` is canonical. The playos-runtime duplicates are removed.

**Steps:**

1. **Reconcile the diverged files.** Diff `playos-runtime/src/ipc_server.c` against `playos-refdistro/src/playos-init/ipc/ipc_server.c`. Port any unique improvements from the playos-runtime version into the playos-init version (or vice versa if the init version is older). Do the same for `ipc_client.c` and `lifecycle_fd.c`.
2. **Verify the merged files compile and pass tests.** Run `cd playos-refdistro/src/playos-init/build && cmake .. && make && ctest` to confirm playos-init still builds and all host tests pass.
3. **Remove IPC source files from playos-runtime.** Delete `playos-runtime/src/ipc_framing.c`, `playos-runtime/src/ipc_server.c`, `playos-runtime/src/ipc_client.c`, `playos-runtime/src/lifecycle_fd.c`, and `playos-runtime/tests/test_ipc_framing.c`.
4. **Update playos-runtime CMakeLists.txt.** Remove the `playos-ipc` library target and the `playos-ipc-tests` executable target. Keep only the protocol XML install target (see S2.5-T7).
5. **Update playos-init CMakeLists.txt** if needed to ensure the IPC source paths didn't reference the playos-runtime copies (they shouldn't — `playos-init/CMakeLists.txt` already references its own `ipc/` directory).
6. **Update `playos-runtime/include/playos-runtime/ipc.h`.** Either remove it (if playos-init owns the header) or replace it with a thin wrapper that `#include`s the canonical header via Buildroot staging. The simplest approach: remove it and note that `playos-runtime` no longer ships IPC — it ships only protocol XML.

**Done when:**
- `playos-runtime/src/` contains no `.c` files (empty or protocol-only).
- `playos-refdistro/src/playos-init/` builds cleanly and all host tests pass.
- No file references the removed playos-runtime IPC sources.

---

### S2.5-T2 — Enforce Version Pinning in `make setup`

**Finding:** `versions.lock` pins exact commit SHAs for all PlayOS components, but `make setup` does `git clone <url>` without checking out the pinned SHA. A `make setup` today vs tomorrow could produce different source trees.

**Steps:**

1. **Parse `versions.lock` in the Makefile.** Add a target or include that reads the `PLAYOS_*_COMMIT` variables from `versions.lock`. Since `versions.lock` uses shell-compatible syntax (`KEY=value`), it can be included directly:

```makefile
# Include version pins (shell-compatible format)
VERSIONS_LOCK := $(CURDIR)/versions.lock
ifneq (,$(wildcard $(VERSIONS_LOCK)))
  # Extract commit SHAs — versions.lock uses VAR=value format
  PLAYOS_INIT_COMMIT := $(shell grep '^PLAYOS_INIT_COMMIT=' $(VERSIONS_LOCK) | cut -d= -f2)
  PLAYOS_COMPOSITOR_COMMIT := $(shell grep '^PLAYOS_COMPOSITOR_COMMIT=' $(VERSIONS_LOCK) | cut -d= -f2)
  PLAYOS_RUNTIME_COMMIT := $(shell grep '^PLAYOS_RUNTIME_COMMIT=' $(VERSIONS_LOCK) | cut -d= -f2)
endif
```

2. **Add `git checkout <SHA>` after each clone in the `setup` target.** After `git clone https://github.com/PlayOS-Foundation/playos-init.git`, add:
```bash
cd "$(CURDIR)/src/playos-init" && git fetch && git checkout $(PLAYOS_INIT_COMMIT)
```
Do the same for playos-compositor. If a SHA is empty (unset), skip the checkout (clone HEAD only).

3. **Add a `--force` flag to `make setup`.** `make setup-force` removes `src/playos-init/` and `src/playos-compositor/` before re-cloning, ensuring a clean checkout at the pinned SHA.

4. **Document the pinning behavior** in the Makefile header comment.

**Done when:**
- `make setup` produces `src/playos-init/` at the commit specified in `versions.lock`.
- `make setup` produces `src/playos-compositor/` at the commit specified in `versions.lock`.
- Running `make setup` twice is idempotent — second run detects existing clones and skips (or warns if SHA mismatch).

---

### S2.5-T3 — Update Sprint-0.md: Board Directory Location

**Finding:** Sprint-0.md's "Expected Files and Directories" section shows `board/` at `playos-refdistro/board/` (repo root), but the actual implementation places it at `br2-external/board/`. The defconfig references `$(BR2_EXTERNAL_PlayOS_PATH)/board/...` which resolves correctly to `br2-external/board/`.

**Decision:** The `br2-external/board/` location is correct for Buildroot conventions. Update the spec, not the code.

**Steps:**

1. Edit `playos-spec/src/sprints/Sprint-0.md` — in the "Expected Files and Directories" section, change:
   ```
   board/
   ├── common/
   └── qemu-x86_64/
   ```
   to:
   ```
   br2-external/board/
   ├── common/
   ├── patches/
   └── qemu-x86_64/
   ```
2. Add a note explaining that board files live under `br2-external/` because Buildroot's `BR2_EXTERNAL` variable resolves paths relative to the external tree, not the repo root.

**Done when:** Sprint-0.md's directory tree matches the actual file layout on disk.

---

### S2.5-T4 — Update Sprint-1.md: Restart Policy

**Finding:** Sprint-1.md's S1-T4 task description says "restart policy (5/10s limit)" and the spec says "restart on clean exit or crash up to a documented retry limit." The actual implementation uses **3 restarts per 60-second window** with a **500ms delay** between restarts. This was a conscious change during Sprint 2, but Sprint-1.md was never updated to reflect the final value.

**Steps:**

1. Edit `playos-spec/src/sprints/Sprint-1.md` — in S1-T4's description, change "restart policy (5/10s limit)" to "restart policy (3 restarts per 60-second window, 500ms delay)".
2. In the "Implementation Guidance → Child process supervision" section, add a note: "The restart policy is defined as constants in `init.h`: `PLAYOS_COMPOSITOR_MAX_RESTARTS=3`, `PLAYOS_COMPOSITOR_WINDOW_S=60`, `PLAYOS_COMPOSITOR_RESTART_DELAY_MS=500`."

**Done when:** Sprint-1.md's documented restart policy matches the constants in `init.h`.

---

### S2.5-T5 — Remove Deprecated `linux.fragment`

**Finding:** `br2-external/board/qemu-x86_64/linux.fragment` still exists on disk. It was replaced by `linux.config` during Sprint 0 build debugging and the defconfig doesn't reference it. It's dead code.

**Steps:**

1. Delete `playos-refdistro/br2-external/board/qemu-x86_64/linux.fragment`.
2. Verify the defconfig doesn't reference it: `grep linux.fragment br2-external/configs/playos_qemu_x86_64_defconfig` — should return nothing.
3. Commit with message: "Remove deprecated linux.fragment (replaced by linux.config in Sprint 0)".

**Done when:** `linux.fragment` no longer exists in the repository.

---

### S2.5-T6 — Implement GPT Partition GUID Search in `mount.c`

**Finding:** `mount.c` implements 5 strategies for data partition discovery, but strategy 4 (GPT partition GUID search) has a TODO placeholder. The spec requires "Search by documented label, UUID, or GPT partition GUID." On physical hardware (Sprint 3), GPT GUIDs may be the only reliable identifier.

**Steps:**

1. Read the current `mount.c` to understand the existing 5-strategy search pattern and where the TODO is.
2. Implement GPT partition GUID search: read the GPT header from the block device, locate the partition entry array, scan for the PlayOS data partition type GUID.
3. The PlayOS data partition type GUID should be defined as a constant (e.g., a random UUID generated for PlayOS, or a well-known one documented in the spec). For now, use a placeholder GUID that can be finalized later, but implement the search logic.
4. If no GPT table exists (e.g., MBR disk), the strategy should log a debug message and fall through to strategy 5 (kernel cmdline UUID).
5. Add a host test (or extend existing tests) that validates the GPT header parsing logic with a mocked GPT disk image.

**Done when:**
- `mount.c` no longer has a TODO for GPT GUID search.
- The search logic is implemented and compiles.
- A host test exercises the GPT parsing (even with a synthetic header).

---

### S2.5-T7 — Wire playos-runtime Buildroot Package to Install Protocol XML

**Finding:** `playos-runtime` Buildroot package is still a Sprint 0 stub (`@true` for both build and install). The `playos-runtime` repo now contains a real protocol XML (`playos-v1.xml`) and previously contained an IPC library (removed in S2.5-T1). The Buildroot package should install the protocol XML into staging so other packages (compositor, future shell) can reference it.

**Steps:**

1. Update `playos-refdistro/br2-external/package/playos-runtime/playos-runtime.mk`:
   - Change `_SITE` to point to the actual cloned source: `$(BR2_EXTERNAL_PlayOS_PATH)/../src/playos-runtime` (requires `make setup` to clone it, or use the existing checkout).
   - Add a build step that does nothing (`@true` — no C code to compile after S2.5-T1).
   - Add an install step that copies `protocols/playos-v1.xml` into `$(STAGING_DIR)/usr/share/playos/protocols/`.
2. Update `Makefile` setup target to clone `playos-runtime` into `src/playos-runtime/` (if not already present).
3. Update `versions.lock` if a `PLAYOS_RUNTIME_COMMIT` pin exists (it does).
4. Update `playos-compositor.mk` to optionally reference the staging protocol XML path instead of bundling its own copy, OR keep the bundled copy and document that the staging copy is the canonical source for downstream consumers.

**Done when:**
- `make setup` clones `playos-runtime` into `src/playos-runtime/`.
- `make qemu-build` installs `playos-v1.xml` into the Buildroot staging directory.
- The compositor build is not broken by this change (it can still use its bundled copy).

---

### S2.5-T8 — Update Sprint-2.md Acceptance Criteria Status

**Finding:** Sprint-2.md has 3 of 10 acceptance criteria still unchecked, all depending on QEMU Buildroot build verification. After the wlroots 0.20 migration (commits `8d10031`, `0a1615e`), the QEMU build should succeed.

**Steps:**

1. Run `make qemu-build` in `playos-refdistro` and verify it completes successfully.
2. If the build passes, run `make qemu-run` (or `scripts/qemu-boot-check.sh`) and verify the compositor starts under playos-init.
3. Update Sprint-2.md:
   - Check off acceptance criteria 2 ("compositor starts under playos-init"), 9 ("Buildroot packages and image integration work end-to-end"), and 10 ("QEMU headless validation remains automated").
   - Update the task status grid if any task was marked "in progress".
   - Update the "Status" line at the top of the document.
4. If the build fails, document the blocker in Sprint-2.md and do NOT mark the criteria as done.

**Done when:**
- Sprint-2.md's acceptance criteria grid matches reality.
- All 10 criteria are either checked (verified) or have a documented blocker.

---

## Implementation Guidance

### Order of execution

1. **T5 first** (remove linux.fragment) — trivial, warms up the workflow.
2. **T2 second** (version pinning) — ensures future clones are reproducible.
3. **T1 third** (IPC unification) — the most complex change, touches two repos.
4. **T6 fourth** (GPT GUID) — isolated change in mount.c.
5. **T7 fifth** (playos-runtime package) — depends on T1 (IPC files removed).
6. **T8 sixth** (verify QEMU build) — depends on all code changes being done.
7. **T3, T4 last** (spec updates) — document reality after all code changes land.

### Atomic commits

Each task should be a separate commit (or small commit group) with a clear message referencing the task ID:

```
S2.5-T2: enforce version pinning in make setup
S2.5-T5: remove deprecated linux.fragment
```

### Do not break the QEMU build

After T1–T7, run `make qemu-build` to confirm nothing regressed. If the build was already broken before this sprint, document that and fix only what this sprint touches.

---

## Verification and Evidence

| Evidence | How it is produced |
|---|---|
| IPC unification proof | `playos-runtime/src/` contains no `.c` files; playos-init host tests pass |
| Version pinning proof | `make setup` followed by `git -C src/playos-init log -1 --format=%H` matches `versions.lock` |
| Spec update proof | `diff` between old and new Sprint-0.md, Sprint-1.md shows corrections |
| Deprecated file removal proof | `linux.fragment` does not exist in the repo |
| GPT GUID proof | `mount.c` has no TODO; host test for GPT parsing passes |
| Protocol staging proof | `playos-v1.xml` exists in Buildroot staging after `make qemu-build` |
| QEMU build proof | `make qemu-build && make qemu-run` produces compositor-ready boot log |

---

## Acceptance Criteria

- [ ] IPC source files exist in only one canonical location (playos-init/ipc/)
- [ ] playos-runtime no longer contains `.c` source files
- [ ] playos-init builds and all host tests pass after IPC unification
- [ ] `make setup` checks out the exact commit SHA from `versions.lock` for playos-init and playos-compositor
- [ ] `make setup` is idempotent — safe to run on an already-set-up tree
- [ ] Sprint-0.md "Expected Files and Directories" matches actual `br2-external/board/` layout
- [ ] Sprint-1.md restart policy text (S1-T4) matches `init.h` constants (3/60s, 500ms)
- [ ] `linux.fragment` is deleted from the repository
- [ ] `mount.c` implements GPT partition GUID search (no TODO placeholder)
- [ ] A host test exercises the GPT parsing logic
- [ ] `playos-runtime` Buildroot package installs protocol XML into staging
- [ ] `make qemu-build` completes successfully after all changes
- [ ] Sprint-2.md acceptance criteria 2, 9, 10 are updated to reflect QEMU build result

---

## Handoff to Sprint 3

Sprint 3 may assume:

- The codebase is clean — no duplicated IPC, no deprecated files, no stale TODOs in the data partition discovery path.
- `make setup` is reproducible — every developer and CI runner gets the exact same source tree.
- The spec documents match reality — file locations, restart policy, and acceptance criteria are accurate.
- GPT partition GUID search works — important for physical hardware where labels may not be set.
- The QEMU build path is verified and the compositor boots under playos-init.

Sprint 3 should not need to fix any of the issues addressed here. If any finding resurfaces, it should be treated as a regression.

---

*Previous: [Sprint 2](Sprint-2.md) | Next: [Sprint 3](Sprint-3.md)*
