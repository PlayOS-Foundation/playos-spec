# Sprint 5.6 — ReposCleanUp

**Goal:** Restore a consistent repository boundary so that every PlayOS component's C source lives in its own repository and the reference distribution contains **no** committed C source. This resolves two structural drifts left over from Sprints 1–5.5: (1) `playos-init` has no repository of its own — its 25 source files are committed directly into `playos-refdistro`; and (2) `playos-shell` source is likewise committed into `playos-refdistro` and is not wired into `make setup`, even though a real `playos-shell` repository already exists and is pushed.

**Primary Outcome:** `playos-init` becomes a first-class repository (created by the maintainer, seeded from the in-tree source). Both `playos-init` and `playos-shell` are removed from the `playos-refdistro` git index, git-ignored, cloned by `make setup` from their own repositories, and pinned by a real commit SHA in `versions.lock`. Buildroot packages continue to build from the `src/` clones unchanged.

**Status:** 🟢 Complete — implemented and verified

**Prerequisites:** Sprint 5.5 complete — the shell renders through Raylib 6.0 and the `playos-shell` repository is pushed at `main`. The maintainer has created (or will create) the empty `PlayOS-Foundation/playos-init` repository.

---

## Why This Sprint Exists

Sprints 0–5.5 accumulated two repository-boundary violations that contradict the project's own rule ("no C code here — all source lives in the component repos") and will compound as more sprints build on these components:

1. **`playos-init` has no repository.** `playos-refdistro/src/playos-init/` is tracked directly in the `playos-refdistro` git index (25 files: `init.c`, `supervisor.c`, `mount.c`, `recovery.c`, `logging.c`, `ipc/*.c`, tests, `CMakeLists.txt`). It has no `.git` and no remote. There is no `PlayOS-Foundation/playos-init` repository yet, even though the `Makefile` `setup` target already tries to clone one and `.gitignore` already lists `src/playos-init`.

2. **`playos-shell` is committed into `playos-refdistro`.** `playos-refdistro/src/playos-shell/` contains 1 392 files tracked in `playos-refdistro` (no `.git`). The real repository exists at `PlayOS-Foundation/playos-shell` and is pushed at `main` (HEAD `1046262…`), and the in-tree copy is a separate, manually-synced snapshot that is currently in sync with it. `make setup` does not clone `playos-shell` at all, and `.gitignore` does not ignore `src/playos-shell`.

3. **`versions.lock` is still dishonest for init.** `PLAYOS_INIT_COMMIT=b7800393…` is annotated `(in monorepo)` and points at a refdistro commit, not a `playos-init` repo commit. `PLAYOS_SHELL_COMMIT` is already correct (`104626206dfd2de59b4c576d3990d43b3b65980b`, the pushed canonical HEAD).

4. **The correct pattern already exists.** `playos-compositor`, `playos-runtime`, and `playos-platform-api` are each: a sibling repository, `src/<name>` git-ignored + untracked, cloned and SHA-checked-out by `make setup`, and built by a `local`-method Buildroot package. `playos-init` and `playos-shell` simply need to be brought onto that same pattern.

This is a **pure cleanup sprint** — no feature, protocol, or UX changes. The user-visible result must be identical.

---

## Start Condition Checklist

- [x] Sprint 5.5 complete; `playos-shell` renders through Raylib 6.0.
- [x] `PlayOS-Foundation/playos-shell` repository is pushed at `main` (verified: HEAD `104626206dfd2de59b4c576d3990d43b3b65980b` == `origin/main`).
- [x] `PlayOS-Foundation/playos-init` repository exists (empty, or with only a README/license). *(Created by the maintainer — not by the implementing agent.)*
- [x] Local checkouts are available: `playos-refdistro` and the sibling `playos-shell` repo under the PlayOS workspace root (`$HOME/playos/`).
- [x] `make setup` / `make qemu-build` are understood and can be run to verify the result.

---

## Decisions Locked for This Sprint

- **`playos-init` becomes its own repository.** The maintainer creates `PlayOS-Foundation/playos-init`. The implementing agent seeds it from the current `playos-refdistro/src/playos-init/` source (excluding build artifacts), pushes it, and records the full HEAD SHA.
- **`playos-shell` canonical source is the sibling repository**, not the in-tree `playos-refdistro/src/playos-shell` copy. The in-tree copy is deleted from the index. If the in-tree copy carries a meaningful uncommitted edit, that edit is applied to the sibling repo and pushed first; otherwise it is discarded.
- **IPC ownership stays with `playos-init`.** The IPC C sources (`ipc/ipc.h`, `ipc_client.c`, `ipc_server.c`, `ipc_framing.c`, `lifecycle_fd.c`) move with `playos-init` into its repository. `playos-runtime` remains protocol-only (unchanged from Sprint 2.5).
- **Buildroot packaging is unchanged.** All five component `.mk` files keep `SITE_METHOD = local` pointing at `$(BR2_EXTERNAL_PlayOS_PATH)/../src/<name>`. `make setup` materializes the clones; the packages build from those clones.
- **No C source is committed in `playos-refdistro` after this sprint.**
- **No new features.** Pure remediation.

---

## Scope

### In Scope

- Seed and push the new `playos-init` repository from the in-tree source
- Untrack `playos-refdistro/src/playos-init` and `playos-refdistro/src/playos-shell` from the refdistro git index
- Add `src/playos-shell` to `playos-refdistro/.gitignore` (init is already listed)
- Wire `playos-shell` into the `Makefile` `setup` target (clone + pinned checkout), mirroring the existing four components
- Update `versions.lock` so `PLAYOS_INIT_COMMIT` points at the new `playos-init` repo HEAD, and remove the `(in monorepo)` annotation (`PLAYOS_SHELL_COMMIT` is already pinned correctly)
- Verify `make setup` clones all five components and `playos-init` / `playos-shell` still build from the clones
- Update docs / repo inventory so no document claims init or shell C source lives in `playos-refdistro`

### Explicitly Out of Scope

- Creating the `playos-init` GitHub repository (maintainer does this)
- New features, protocol changes, or UX changes
- Changing the Buildroot `local` site method or package structure
- Re-homing the IPC protocol into `playos-runtime` or a new repo (deferred until a consumer actually needs it)
- CI pipeline changes
- Sprint 6 storage / game-discovery work

---

## Required Repository Changes

| Repo | Required work |
|---|---|
| `playos-init` | **NEW** — seeded from `playos-refdistro/src/playos-init/` (source + tests + `CMakeLists.txt` + `.gitignore`), pushed to `main` |
| `playos-shell` | Verify no drift vs. canonical (already in sync); ensure `main` is the pushed canonical state |
| `playos-refdistro` | `git rm -r src/playos-init src/playos-shell`; add `src/playos-shell` to `.gitignore`; wire `playos-shell` into `Makefile` `setup`; fix `versions.lock`; update `AGENTS.md` |
| `playos-spec` | Add `Sprint-5.6.md`; update repo inventory (`architecture.md`) to list `playos-init`; update cross-link footers |

---

## Expected Files and Directories

### `playos-init` (NEW repository, after seeding)

```text
.gitignore               ← carried over from the in-tree source
CMakeLists.txt
include/playos-init/     ← init.h, ipc_handler.h, mount.h, recovery.h, supervisor.h
ipc/                     ← ipc.h, ipc_client.c, ipc_server.c, ipc_framing.c, lifecycle_fd.c
src/                     ← child_process.c, init.c, ipc_handler.c, logging.c, main.c, mount.c, recovery.c, shutdown.c, supervisor.c
tests/                   ← host + ipc test sources
```

> `build/`, `CMakeFiles/`, `CMakeCache.txt`, and `cmake_install.cmake` must **not** be committed.

### `playos-refdistro`

```text
.gitignore                ← UPDATE: add `src/playos-shell` to the "Cloned source dependencies" block
Makefile                  ← UPDATE: add PLAYOS_SHELL_COMMIT var + playos-shell clone block in `setup`
versions.lock             ← UPDATE: real SHA for INIT only (from S5.6-T1); remove the `(in monorepo)` annotation
AGENTS.md                 ← UPDATE: IPC Sources note points at the playos-init repo (cloned to src/playos-init)
src/playos-init/          ← REMOVED from index + working tree (re-created by make setup)
src/playos-shell/         ← REMOVED from index + working tree (re-created by make setup)
```

### `playos-spec`

```text
src/sprints/Sprint-5.6.md     ← NEW: this document
src/sprints/Sprint-5.5.md     ← UPDATE: footer "Next" → Sprint 5.6
src/sprints/Sprint-6.md       ← UPDATE: footer "Previous" → Sprint 5.6
src/architecture.md           ← UPDATE: add playos-init to the repository inventory
```

---

## Agent Task Breakdown

Every task below is independently checkable.

### Task Status Grid

| Task ID | Task | Primary repo | Status | Notes / evidence |
|---|---|---|---|---|
| S5.6-T1 | Seed and push the `playos-init` repository | `playos-init` | done | Cloned empty repo, copied in-tree source, committed, pushed (`3a89f09f…`) |
| S5.6-T2 | Reconcile and confirm the `playos-shell` canonical SHA | `playos-shell` | done | `1046262…` confirmed current; in-tree copy already in sync |
| S5.6-T3 | Untrack `src/playos-init` from refdistro | `playos-refdistro` | done | `git rm -r src/playos-init`; already in `.gitignore` |
| S5.6-T4 | Untrack `src/playos-shell` from refdistro and gitignore it | `playos-refdistro` | done | `git rm -r src/playos-shell`; added to `.gitignore` |
| S5.6-T5 | Wire `playos-shell` into `make setup` | `playos-refdistro` | done | Added `PLAYOS_SHELL_COMMIT` + clone block, mirroring the other four |
| S5.6-T6 | Pin real SHAs and clean `versions.lock` | `playos-refdistro` | done | Removed the `(in monorepo)` annotation (shell already pinned) |
| S5.6-T7 | Verify `make setup` + build from extracted repos | `playos-refdistro` | done | All five clones present; init + shell build |
| S5.6-T8 | Docs and repo-inventory reconciliation | `playos-spec`, `playos-refdistro` | done | No doc claims C source lives in refdistro |

---

### S5.6-T1 — Seed and Push the `playos-init` Repository

**Finding:** The only copy of `playos-init` source is tracked in `playos-refdistro/src/playos-init/` (25 files, no `.git`, no remote). The maintainer has created the empty `PlayOS-Foundation/playos-init` repository. The `Makefile` `setup` target already attempts to clone it, so the repo must exist and contain the source before `make setup` can succeed.

**Steps:**

1. Clone the (empty) repository to a scratch location:
   ```sh
   git clone https://github.com/PlayOS-Foundation/playos-init.git /tmp/playos-init-seed
   ```
2. Copy the source from the in-tree copy into the clone, preserving the directory layout:
   ```sh
   rsync -a --exclude build --exclude CMakeFiles --exclude CMakeCache.txt \
     --exclude cmake_install.cmake \
     playos-refdistro/src/playos-init/ /tmp/playos-init-seed/
   ```
   (If `rsync` is unavailable, use `cp -a` and remove the generated `build/` directory afterward.)
3. Verify the clone's own `.gitignore` (carried over from the in-tree source) excludes `build/` and other generated artifacts; adjust if it does not.
4. Stage, commit, and push:
   ```sh
   cd /tmp/playos-init-seed
   git add -A
   git commit -m "S5.6-T1: seed playos-init from refdistro source"
   git push -u origin main
   ```
5. Record the full HEAD SHA:
   ```sh
   git rev-parse HEAD
   ```

**Done when:**
- `PlayOS-Foundation/playos-init` contains the same source layout as the former in-tree copy (minus build artifacts).
- The repo is pushed to `main`.
- A full 40-char HEAD SHA is recorded for use in `versions.lock`.

---

### S5.6-T2 — Reconcile and Confirm the `playos-shell` Canonical SHA

**Finding:** The real `playos-shell` repository (sibling checkout `$HOME/playos/playos-shell`) is pushed and clean at HEAD `104626206dfd2de59b4c576d3990d43b3b65980b` (`main` == `origin/main`). The in-tree `playos-refdistro/src/playos-shell/` copy is already byte-identical to the sibling repo — the `rcore_playos.c` non-blocking render-loop fix was committed to `playos-shell` as `1046262` and synced into `playos-refdistro` as `257ac52`. `versions.lock` already points at `104626206dfd2de59b4c576d3990d43b3b65980b`. This task is therefore a **verification no-op** unless a fresh diff surfaces drift.

**Steps:**

1. Confirm the sibling repo is clean and in sync with its remote:
   ```sh
   git -C playos-shell status --short
   git -C playos-shell rev-parse origin/main
   ```
2. Diff the in-tree copy against the sibling repo to confirm no drift:
   ```sh
   diff -ru playos-shell playos-refdistro/src/playos-shell | head -200
   ```
3. Record the canonical full SHA `104626206dfd2de59b4c576d3990d43b3b65980b` for `versions.lock`. If a diff surfaces a meaningful edit, apply it to the sibling repo, commit, and push first, then use the new `origin/main` SHA.

**Done when:**
- `playos-shell` `main` and `origin/main` are in sync.
- The in-tree copy is confirmed byte-identical to the sibling repo (no drift).
- A single canonical full SHA `104626206dfd2de59b4c576d3990d43b3b65980b` is recorded for `versions.lock`.

---

### S5.6-T3 — Untrack `src/playos-init` from `playos-refdistro`

**Finding:** `src/playos-init` is tracked in `playos-refdistro` (25 files). `.gitignore` already lists `src/playos-init` (line 18), but ignore rules do not untrack files that are already in the index. The directory must be removed from both the index and the working tree so `make setup` will clone it fresh.

**Steps:**

1. Remove from index **and** working tree (this is intentional — `make setup` will re-materialize it):
   ```sh
   cd playos-refdistro
   git rm -r src/playos-init
   ```
2. Confirm `.gitignore` still contains `src/playos-init` (no change needed).
3. Commit:
   ```sh
   git commit -m "S5.6-T3: untrack playos-init source from refdistro monorepo"
   ```
4. Verify nothing remains tracked:
   ```sh
   git ls-files src/playos-init   # expected: no output
   ```

**Done when:**
- `git ls-files src/playos-init` returns nothing.
- `src/playos-init/` no longer exists in the working tree.
- The commit is present with the task ID in its message.

---

### S5.6-T4 — Untrack `src/playos-shell` from `playos-refdistro` and Gitignore It

**Finding:** `src/playos-shell` is tracked in `playos-refdistro` (1 392 files) and is **not** in `.gitignore`. It must be removed from the index and working tree, and `src/playos-shell` must be added to `.gitignore` so `make setup`'s clone is not re-added.

**Steps:**

1. Remove from index and working tree:
   ```sh
   cd playos-refdistro
   git rm -r src/playos-shell
   ```
2. Add `src/playos-shell` to `.gitignore` under the existing "Cloned source dependencies" block (after the `src/playos-runtime` line):
   ```text
   src/playos-shell
   ```
3. Commit:
   ```sh
   git commit -m "S5.6-T4: untrack playos-shell source from refdistro monorepo"
   ```
4. Verify:
   ```sh
   git ls-files src/playos-shell   # expected: no output
   git check-ignore src/playos-shell && echo "ignored"
   ```

**Done when:**
- `git ls-files src/playos-shell` returns nothing.
- `.gitignore` lists `src/playos-shell`.
- The commit is present with the task ID in its message.

---

### S5.6-T5 — Wire `playos-shell` into `make setup`

**Finding:** The `Makefile` parses `PLAYOS_INIT_COMMIT`, `PLAYOS_COMPOSITOR_COMMIT`, `PLAYOS_RUNTIME_COMMIT`, and `PLAYOS_PLATFORM_API_COMMIT` (lines 31–34) and clones those four repos in `setup` (lines 55–86). It has **no** `PLAYOS_SHELL_COMMIT` and **no** shell clone, so `make setup` cannot produce `src/playos-shell`.

**Steps:**

1. In the version-pins block, add a `PLAYOS_SHELL_COMMIT` variable mirroring the existing four (e.g. after line 34):
   ```makefile
   PLAYOS_SHELL_COMMIT := $(shell grep -s '^PLAYOS_SHELL_COMMIT=' $(VERSIONS_LOCK) 2>/dev/null | cut -d= -f2- | xargs)
   ```
2. In the `setup` target, add a `playos-shell` clone block mirroring the `playos-init` block (lines 55–62), with the same "clone if missing, then checkout the pinned SHA if set" guard:
   ```makefile
   @if [ ! -d "$(CURDIR)/src/playos-shell" ]; then \
       echo "  -> playos-shell..."; \
       git clone https://github.com/PlayOS-Foundation/playos-shell.git "$(CURDIR)/src/playos-shell"; \
       if [ -n "$(PLAYOS_SHELL_COMMIT)" ]; then \
           cd "$(CURDIR)/src/playos-shell" && \
           git fetch origin && git checkout "$(PLAYOS_SHELL_COMMIT)"; \
       fi; \
   fi
   ```
3. Keep the ordering consistent (init, compositor, runtime, platform-api, shell, or any stable order).

**Done when:**
- `make setup` clones `playos-shell` into `src/playos-shell` and checks out the pinned SHA.
- The new block is structurally identical to the existing four.

---

### S5.6-T6 — Pin Real SHAs and Clean `versions.lock`

**Finding:** `versions.lock` currently has:
- `PLAYOS_INIT_COMMIT=b7800393c9466609facf2c5c57dacbe3c9bb0c30  # sprint-5: supervisor updates (in monorepo)`
- `PLAYOS_SHELL_COMMIT=104626206dfd2de59b4c576d3990d43b3b65980b  # sprint-5.5: raylib 6.0 backend + non-blocking render loop (fixes frame-2 deadlock)`

Only the init entry is now wrong: its SHA refers to a refdistro commit (not a `playos-init` repo commit), so it must be replaced after S5.6-T1 seeds the repo. The shell SHA is already correct and pushed, so no change is required there.

**Steps:**

1. Replace `PLAYOS_INIT_COMMIT` with the full SHA recorded in **S5.6-T1**, and remove the `(in monorepo)` annotation:
   ```text
   PLAYOS_INIT_COMMIT=<playos-init-head-sha>  # sprint-5.6: extracted to own repo
   ```
2. Confirm `PLAYOS_SHELL_COMMIT` already equals the canonical SHA from **S5.6-T2** (`104626206dfd2de59b4c576d3990d43b3b65980b`) — no change needed.
3. Confirm no entry still carries `(in monorepo)`, `(LOCAL — push pending)`, or a branch name:
   ```sh
   grep -nE 'monorepo|push pending|latest|main' versions.lock   # expected: no matches (except comments that are intentional)
   ```

**Done when:**
- Both SHAs are non-empty, 40-char, and resolve in their respective repositories.
- No stale annotation remains in `versions.lock`.

---

### S5.6-T7 — Verify `make setup` and the Build from Extracted Repos

**Finding:** After T3–T6, `make setup` should clone all five components, and `playos-init` / `playos-shell` should build from those clones. The Buildroot `.mk` files are unchanged (`SITE_METHOD = local`), so this verifies the repository-boundary change without touching packaging.

**Steps:**

1. Re-run setup (a fresh `src/` is expected after T3/T4):
   ```sh
   cd playos-refdistro
   make setup
   ```
2. Confirm all five clones are present and at the pinned SHAs:
   ```sh
   for d in playos-init playos-compositor playos-runtime playos-platform-api playos-shell; do
     printf '%s: ' "$d"
     git -C "src/$d" rev-parse --short HEAD
   done
   ```
3. Build the two affected packages (faster than a full image; use the QEMU output dir, or `ally` for the device path):
   ```sh
   make -C buildroot BR2_EXTERNAL="$PWD/br2-external" O="$PWD/output/qemu" \
     playos-init-rebuild playos-shell-rebuild
   ```
   (Alternatively run `make qemu-build` for the full end-to-end check, as in prior sprints.)
4. Confirm the refdistro index carries no component source:
   ```sh
   git ls-files src/   # expected: no *.c/*.h under src/ (all five now gitignored clones)
   ```

**Done when:**
- `make setup` materializes all five `src/` clones at their pinned SHAs.
- `playos-init` and `playos-shell` rebuild successfully from the clones.
- `git ls-files src/` shows no committed component source.

---

### S5.6-T8 — Docs and Repo-Inventory Reconciliation

**Finding:** Several documents still describe the pre-cleanup reality. `playos-spec/src/architecture.md` (or its repository inventory) omits `playos-init` as a first-class repository. `playos-refdistro/AGENTS.md`'s "IPC Sources" note says IPC C sources "live at `src/playos-init/ipc/`", which now means the clone, not the monorepo.

**Steps:**

1. In `playos-spec/src/architecture.md` (or wherever the repository list lives), add `playos-init` alongside `playos-compositor`, `playos-runtime`, `playos-platform-api`, and `playos-shell` as a first-class component repository.
2. Update `playos-refdistro/AGENTS.md` so the "IPC Sources" note reads that the IPC sources live in the `playos-init` repository (cloned to `src/playos-init/ipc/` by `make setup`), not committed in `playos-refdistro`.
3. Add this sprint to the footer chain (see this document's footer, and the Sprint-5.5 / Sprint-6 footer edits below).
4. Confirm no remaining doc claims init or shell C source is committed in `playos-refdistro`:
   ```sh
   grep -rnE 'in monorepo|in-tree|committed (directly|into) playos-refdistro' \
     playos-refdistro/AGENTS.md playos-spec/src/architecture.md
   ```

**Done when:**
- `playos-init` is listed as a first-class repository.
- `playos-refdistro/AGENTS.md` no longer implies init/shell C source is committed in the distribution repo.
- Footer links point Previous/Next through Sprint 5.6.

---

## Implementation Guidance

### Order of execution

1. **T1 first** (seed + push `playos-init`) — everything downstream needs the repo to exist.
2. **T2 second** (reconcile + confirm `playos-shell` SHA) — locks the canonical shell state.
3. **T3, T4 next** (untrack both in-tree copies) — removes the monorepo source; safe because the repos are pushed.
4. **T5 fifth** (wire shell into `make setup`) — no dependency on T3/T4 beyond intent, but do it before verifying.
5. **T6 sixth** (pin SHAs in `versions.lock`) — depends on T1/T2 for the real SHAs.
6. **T7 seventh** (verify) — depends on all code/index changes.
7. **T8 last** (docs) — document reality after the code lands.

### Atomic commits

Each task is a separate commit (or small group) referencing the task ID:

```
S5.6-T1: seed playos-init repo from refdistro source
S5.6-T3: untrack playos-init source from refdistro monorepo
S5.6-T4: untrack playos-shell source from refdistro monorepo
S5.6-T5: wire playos-shell into make setup
S5.6-T6: pin real init/shell SHAs in versions.lock
S5.6-T8: update repo inventory and AGENTS.md for extracted repos
```

### Do not break the build

After T1–T7, run `make qemu-build` (or at minimum the targeted `playos-init-rebuild` / `playos-shell-rebuild`) to confirm nothing regressed. The Buildroot `.mk` files must remain unchanged — the only `Makefile` change is the added shell clone block. If a build was already broken before this sprint, document it and fix only what this sprint touches.

### Reversibility note

`git rm -r` removes tracked files from both the index and the working tree. Before running it, confirm the corresponding source has been pushed to its repository (T1 for init, T2 for shell). The pushed repos are the recovery mechanism; the working-tree copies are intentionally disposable.

---

## Verification and Evidence

| Evidence | How it is produced |
|---|---|
| `playos-init` repo proof | `git ls-remote https://github.com/PlayOS-Foundation/playos-init.git main` resolves; repo contains `src/init.c` etc. |
| `playos-shell` canonical SHA proof | `git -C playos-shell rev-parse origin/main` == the SHA in `versions.lock` |
| Init untrack proof | `git ls-files src/playos-init` in `playos-refdistro` returns nothing |
| Shell untrack proof | `git ls-files src/playos-shell` returns nothing; `.gitignore` lists `src/playos-shell` |
| `make setup` proof | `make setup` clones all five `src/` repos at pinned SHAs |
| Build proof | `playos-init-rebuild` and `playos-shell-rebuild` succeed from the clones |
| `versions.lock` proof | both `PLAYOS_INIT_COMMIT` / `PLAYOS_SHELL_COMMIT` are non-empty 40-char SHAs with no stale annotation |
| Docs proof | `playos-init` listed in the repo inventory; `AGENTS.md` no longer claims C source lives in `playos-refdistro` |

---

## Acceptance Criteria

- [x] `PlayOS-Foundation/playos-init` exists, is seeded from the in-tree source (minus build artifacts), and is pushed to `main`
- [x] `PlayOS-Foundation/playos-shell` `main` == `origin/main`, and its full SHA is recorded
- [x] `git ls-files src/playos-init` and `git ls-files src/playos-shell` in `playos-refdistro` both return nothing
- [x] `.gitignore` lists `src/playos-shell` (and still lists `src/playos-init`)
- [x] `make setup` clones all five components into `src/` and checks out the pinned SHAs
- [x] `playos-init` and `playos-shell` rebuild successfully from the clones
- [x] `versions.lock` has real, non-empty, 40-char SHAs for both `PLAYOS_INIT_COMMIT` and `PLAYOS_SHELL_COMMIT`, with no `(in monorepo)` / `(LOCAL — push pending)` annotations
- [x] `playos-refdistro` contains no committed C source under `src/`
- [x] `playos-init` is listed as a first-class repository in the spec's repository inventory
- [x] `playos-refdistro/AGENTS.md` no longer describes init/shell C source as committed in the distribution repo

---

## Handoff to Sprint 6

Sprint 6 may assume:

- Every component's C source lives in its own repository (`playos-init`, `playos-compositor`, `playos-runtime`, `playos-platform-api`, `playos-shell`).
- `playos-refdistro` contains no committed C source — only Buildroot packaging, configs, board files, and the developer `Makefile`.
- `make setup` reproducibly clones all five components at pinned SHAs from `versions.lock`.
- Buildroot still builds each component from its `src/` clone via the unchanged `SITE_METHOD = local` packages.
- The IPC sources live in the `playos-init` repository (cloned to `src/playos-init/ipc/`), and `playos-runtime` remains protocol-only.

Sprint 6 (persistent storage and game discovery) should not need to touch repository boundaries; if any in-tree C source reappears, treat it as a regression from this sprint.

---

*Previous: [Sprint 5.5](Sprint-5.5.md) | Next: [Sprint 6](Sprint-6.md)*
