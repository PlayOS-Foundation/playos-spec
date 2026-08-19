# Sprint 21 — Multiple Local User Profiles (Post-MVP)

**Goal:** Produce a written plus/minus and feasibility assessment for **console-style local user profiles** — multiple on-device PlayOS users whose saves, cache, settings, and screenshots are isolated from each other, analogous to PlayStation console profiles. This sprint records the assessment and scopes the design; no profile implementation is built.

**Primary Outcome:** A decision-ready `Sprint-21.md` record that (a) defines what "multiple local users" means for PlayOS, (b) recommends a path-scoped profile approach layered on the Sprint 12 sandbox, (c) documents the storage, launch, and shell integration, (d) resolves the `ideas.md` "multiple interactive users" wording conflict, and (e) scopes Sprint 12 as the isolation foundation.

**Status:** 🟡 Post-MVP — assessment/design only; not scheduled. No implementation work is approved.

**Prerequisites:** MVP stable (Sprint 15–16); persistent storage and game discovery (Sprint 6); `playos-init` sandbox with data-driven path policy (Sprint 12); single `playos-game` identity documented in the security model (Sprint 12).

---

## Why This Sprint Exists

A shared family handheld has exactly the same profile problem a console does: each person wants their own saves, settings, screenshots, and progress, without seeing or overwriting anyone else's. PlayOS currently has a single implicit user (`playos-game`) and a single per-game save/cache path keyed only by `PLAYOS_GAME_ID`. This sprint assesses how to add **many isolated local profiles** without turning PlayOS into a general-purpose multi-login Linux system, and it aligns Sprint 12 so the sandbox is ready to accept a profile dimension later.

The scope is deliberately **local** profiles only. Parental controls, PIN, and per-profile game libraries are separate future features; **PlayOS Network account linking** (the online identity that unlocks Marketplace and Online Gaming Services) is a follow-on layered on this local profile id.

---

## Assessment Inputs

The assessment rests on the following authoritative facts, verified against source and spec:

- **Storage paths are keyed only by game id.** `playos-platform-api/src/playos_storage.c:36-41` builds `/data/saves/%s`, `/data/cache/%s`, and related paths from `PLAYOS_GAME_ID` only. `include/playos/playos_storage.h:26-45` documents per-game paths, and `:81-90` returns `/data/games` as the shared game-install root.
- **Launch does not yet drop privileges.** `playos-init/src/supervisor.c:762-806` spawns the game child with `setsid()` + environment + `execl()` and **no** setuid/setgid/`PR_SET_NO_NEW_PRIVS`. The env set at `:773-782` already includes `PLAYOS_GAME_ID`, `PLAYOS_INSTALL_PATH`, `PLAYOS_SAVE_PATH`, `PLAYOS_CACHE_PATH`, `WAYLAND_DISPLAY`, `PLAYOS_LIFECYCLE_FD`, and `PLAYOS_LAUNCH_TOKEN` — the natural injection point for a future `PLAYOS_PROFILE_ID`. `security-model.md:228-235` records this pre-Sprint-12 gap.
- **IPC auth is identity-based, not profile-based.** `playos-init/ipc/ipc_server.c:83-88` accepts peers with GID 1000 (`playos-trusted`) or uid 0. Profiles do not change trusted-component auth; they are a data boundary, not a control boundary.
- **Shell discovery is global.** `playos-shell/src/screen_library.c:379-424` scans `/data/games` with a readdir and manifest validation (`:154-265` requires `executable` + `architecture`). Game installs stay shared, so discovery is unchanged.
- **Security model already assumes one identity.** `security-model.md:58-65` privileges a single `playos-game`; `:119` chowns game data to `playos-game:playos-game`; `:203-212` lists Landlock allowed paths; `:337` names user namespaces as post-MVP hardening. Sprint 12 makes this real.
- **A `/data/profiles/` placeholder already exists but is unused.** `architecture.md:363` documents `saves/<game-id>/ profiles/, autosaves/, settings/` and `:371` shows `profiles/`; `partition-layout.md:88` and `:96` reserve `/data/profiles/`. The path is in the design, not yet in `playos_storage.c`.
- **`post-mvp.md` already has a profiles block.** `post-mvp.md:160-163` lists *Multiple Local User Profiles* with a `/data/saves/<profile>/<game-id>/` layout that this sprint corrects to `/data/profiles/<pid>/...`.
- **`ideas.md` wording conflict.** `ideas.md:104-105` lists "multiple interactive users" as never-planned. That entry means multiple interactive **Linux** login users, which remains out of scope; console-style local profiles are separate and planned post-MVP.

---

## Feasibility Assessment

### Verdict

**Feasible post-MVP; moderate effort; low architectural risk.** PlayOS can support console-style local profiles by keeping a single Linux identity (`playos-game`) and adding a profile id to the storage path and sandbox allowlist. This is a data-path + shell feature layered on the Sprint 12 isolation work — not a hard blocker and not a re-architecture. The recommended approach is **path-scoped profiles** (single uid + profile-scoped Landlock), with **uid-per-profile** deferred to later hardening only if profiles must become hard security boundaries (e.g. parental controls).

### Interpretation

"Many local users isolated from each other, like PlayStation" means **console-style local profiles**, not multiple interactive Linux login users. PlayOS keeps one unprivileged runtime identity and isolates profile *data* rather than creating a Unix account per profile. This preserves the Sprint 12 trust model and avoids reopening `ideas.md`'s never-planned "multiple interactive users" item.

### Approach comparison

| Approach | Identity | Isolation mechanism | Effort | Risk | Recommendation |
|---|---|---|---|---|---|
| **A — path-scoped profiles** | Single `playos-game` uid + `PLAYOS_PROFILE_ID` env | Profile-scoped Landlock allowlist + per-profile storage paths | Low–moderate | Low | **Recommended** |
| **B — uid-per-profile** | One Linux uid per profile | OS-level user separation | High | Medium | Defer to hardening |

Approach A is recommended because it changes one path-construction function plus a sandbox prefix, not the trusted-component boundary, IPC auth, or the compositor. Approach B is a real security boundary but costs a second identity model and per-profile ownership, which is unjustified until a feature (parental controls, account-locked content) demands it.

### What data is isolated

| Data | Scope | Notes |
|---|---|---|
| Game saves | Per-profile | `/data/profiles/<pid>/saves/<game-id>/` |
| Game cache | Per-profile | `/data/profiles/<pid>/cache/<game-id>/` |
| PlayOS settings | Per-profile | `/data/profiles/<pid>/settings/` |
| Screenshots | Per-profile | `/data/profiles/<pid>/screenshots/` |
| Account credentials / tokens | Per-profile (when linked) | `/data/profiles/<pid>/auth/` — encrypted, platform-managed, out of the game sandbox |
| Game installs | Shared | `/data/games/` unchanged |
| Wi-Fi / network config | System-wide | Not per-profile |
| Logs | System-wide (per-profile optional) | `/data/logs/` |

Game installs remain shared by design: this mirrors consoles, where all profiles can launch installed titles but each has separate saves and settings.

### Plus / Minus

| Dimension | Plus | Minus |
|---|---|---|
| Product | PlayStation-style local profiles on a shared family handheld | More UX surface: profile picker, switcher, per-profile settings screens |
| Architecture | Layers on Sprint 12's data-driven sandbox; single uid means no IPC/auth/compositor rework | Adds a profile-id dimension to every storage path and env contract |
| Isolation | Profile-scoped Landlock gives real save/cache isolation between profiles | Game installs are shared by design; not a hard security boundary between mutually-untrusting profiles |
| Migration | Legacy `/data/saves/<game-id>` maps cleanly to a default profile | One-time migration code plus a read fallback |
| Effort | Low–moderate; mostly storage-path derivation + shell UI | Per-profile settings/screenshots must be threaded through the versioned `libplayos` C ABI |

### Integration design (how it fits the existing lifecycle)

1. **Storage:** `playos_storage.c/.h` derive paths from `PLAYOS_GAME_ID` **and** `PLAYOS_PROFILE_ID`; keep the existing functions returning the active profile's paths, and add profile-explicit variants. This is an additive, versioned C ABI change.
2. **Launch:** `playos-init` loads the active profile at boot, sets `PLAYOS_PROFILE_ID` and profile-scoped `PLAYOS_SAVE_PATH`/`PLAYOS_CACHE_PATH` into the game env (`supervisor.c:773-782`), and builds the Landlock allowlist with the profile prefix.
3. **Shell:** add profile selection at boot, a user switcher, and per-profile settings; discovery of shared game installs (`/data/games`) is unchanged.
4. **IPC/auth:** unchanged. Profiles are a data boundary, not a trusted-component boundary.

### Migration

- Introduce `/data/profiles/<pid>/...`; treat legacy `/data/saves/<game-id>` as the **default profile** (`<pid>` = `default`) with a one-time move or read fallback, so existing saves are not orphaned.
- Do **not** migrate game installs; they stay shared.

### Spec disambiguation

- **`ideas.md:104-105`:** record here that "multiple interactive users" means multiple interactive **Linux** users, which stays never-planned. Console-style local profiles are a distinct, post-MVP feature. `ideas.md` is **not** edited (spec policy).
- **Overloaded "profiles":** `/data/saves/<game-id>/profiles/` (game-internal save slots) is a different concept from the top-level `/data/profiles/` (PlayOS users). The post-MVP entry and future storage docs must use the top-level path to avoid collision.

### Sequencing

Sprint 12 must land **first**, because it makes the sandbox data-driven and parameterized by launch identity. Sprint 21 then adds the profile dimension by changing one path-construction function, not the enforcement logic. PIN/parental-controls/uid-per-profile remain later.

### Account linking — PlayOS Network (online identity)

The local profile is the offline root of identity. A **PlayOS Network account** is the online identity in `playos-cloud`; a local profile can be **linked** to exactly one Network account, and the link is what unlocks **PlayOS Marketplace** and **Online Gaming Services**.

**Self-hosting:** PlayOS Network and `playos-cloud` services follow the `playos-cloud` **self-hostable-by-design** mandate — every hosted feature has a documented self-hosted deployment, and no feature may require a single central provider. This applies to accounts/identity, cloud saves, matchmaking, and any Marketplace online services.

| Layer | What it is | Unlocks |
|---|---|---|
| Local profile | Offline identity on device (`PLAYOS_PROFILE_ID`) | Local saves/settings/screenshots; offline play |
| PlayOS Network account | Online identity in `playos-cloud` (OAuth2 or PlayOS account service) | Marketplace access, cloud saves, multiplayer/matchmaking, friends/presence |
| Link | Local profile ↔ one Network account (stored per-profile) | Account-scoped entitlements and online services for that profile |

**Feasibility:** feasible post-MVP as a follow-on to this sprint; it does **not** require multiple Linux users. It adds `playos-cloud` services plus per-profile credential storage, so it is more work than the local-profile layer alone.

**Security boundary:** the raw account token must never reach a game. The platform holds credentials in a trusted service and issues games short-lived, scoped session tickets per title/service. This keeps the Sprint 12 game sandbox intact: a linked account does not make the game trusted. Credential storage is per-profile (`/data/profiles/<pid>/auth/`, encrypted at rest) and out of the game's Landlock allowlist.

**Implications for the isolation approach:** account-linked profiles hold credentials, which is the first strong argument for later hardening (Approach B uid-per-profile or a platform keychain). It is **not** required for v1: path-scoped profiles plus a trusted credential service outside the game sandbox are sufficient.

**Sequencing:** Wi-Fi (Sprint 16) → local profiles (Sprint 21) → PlayOS Network account linking + `playos-cloud` → Marketplace (Sprint 19) and Online Gaming Services consume the linked identity. Marketplace v1 can stay free-content/device-local (Sprint 19) and add account-scoped entitlements only after linking exists.

---

## Scope

### In Scope (this sprint)

- This `Sprint-21.md` document.
- The feasibility verdict, Approach A vs B recommendation, plus/minus, isolation matrix, integration design, migration, and spec disambiguation above.
- The Sprint 12 alignment additions recorded below (made as part of this sprint's spec output).

### Explicitly Out of Scope / Not Planned

- Any storage, init, or shell code changes for profiles.
- Multiple interactive Linux login users (never-planned; `ideas.md:104-105`).
- PIN, parental controls, per-profile game libraries, uid-per-profile, and PlayOS Network account linking (the online identity layer is a follow-on, not part of this local-profile sprint).
- Editing `ideas.md` or rewriting any ADR.

---

## Required Repository Changes

| Repo | Required work |
|---|---|
| `playos-spec` | Add `Sprint-21.md`; align `Sprint-12.md` for isolation foundations; update `SUMMARY.md`, `post-mvp.md`, and `sprints/roadmap.md` |
| *(none else)* | No implementation repositories change in this sprint |

---

## Expected Files and Directories

```text
playos-spec/src/sprints/Sprint-21.md   # ADD: this assessment/design
playos-spec/src/sprints/Sprint-12.md   # UPDATE: data-driven sandbox + single-identity foundations
playos-spec/src/SUMMARY.md             # UPDATE: link title
playos-spec/src/post-mvp.md            # UPDATE: profiles entry (path-corrected)
playos-spec/src/sprints/roadmap.md     # UPDATE: sprint-plan row
```

---

## Agent Task Breakdown

### Task Status Grid

| Task ID | Task | Primary repo | Status | Notes / evidence |
|---|---|---|---|---|
| S21-T1 | Define "multiple local users" and select the isolation approach | `playos-spec` | not started | `ideas.md:104-105`, `security-model.md:58-65` |
| S21-T2 | Design profile-scoped storage and launch env | `playos-spec` | not started | `playos_storage.c:36-41`, `supervisor.c:773-782` |
| S21-T3 | Design shell profile selection/switcher and settings | `playos-spec` | not started | `screen_library.c:379-424`, shared `/data/games` |
| S21-T4 | Align Sprint 12 for isolation foundations | `playos-spec` | not started | Landlock ruleset data-driven; single uid |

### S21-T1 — Define "multiple local users" and select the approach

- Record that PlayOS profiles are **console-style local profiles**, not multiple interactive Linux login users (disambiguating `ideas.md:104-105` without editing it).
- Recommend **Approach A (path-scoped profiles)** over Approach B (uid-per-profile), with the single-`playos-game` identity preserved.

**Done when:** the sprint states the interpretation and the recommended approach with a comparison table.

### S21-T2 — Design profile-scoped storage and launch env

- Confirm storage paths are keyed only by `PLAYOS_GAME_ID` (`playos_storage.c:36-41`) and that `/data/games` is shared (`playos_storage.h:81-90`).
- Design `/data/profiles/<pid>/saves|screenshots|cache|settings/`, with game installs shared and network/log config system-wide.
- Design the additive `PLAYOS_PROFILE_ID` + profile-scoped `PLAYOS_SAVE_PATH`/`PLAYOS_CACHE_PATH` env at `supervisor.c:773-782`, and a legacy `/data/saves/<gid>` → default-profile migration.

**Done when:** the sprint records the storage layout, env contract, and migration with source citations.

### S21-T3 — Design shell profile selection and switcher

- Confirm discovery is global (`screen_library.c:379-424`) and stays shared for game installs.
- Design boot-time profile selection, a user switcher, and per-profile settings screens.

**Done when:** the sprint records the shell integration without requiring a discovery change.

### S21-T4 — Align Sprint 12 for isolation foundations

- Add to Sprint 12: keep a single `playos-game` uid (no per-user uid now); parameterize the sandbox path policy by launch identity; build the Landlock ruleset from launch-time variables (game-id now, profile-id later); and keep Landlock path construction data-driven.
- Mark multi-user local profiles as explicitly out of Sprint 12's scope and deferred to Sprint 21.

**Done when:** `Sprint-12.md` contains those four minimal alignment edits and no structural rewrite.

---

## Verification and Evidence

| Evidence | How it is produced |
|---|---|
| Assessment recorded | `Sprint-21.md` present with verdict, approach comparison, plus/minus, isolation matrix, integration design, migration, and disambiguation |
| Approach selected | Approach A recommended over Approach B with rationale |
| Storage/env design grounded | `playos_storage.c:36-41`, `supervisor.c:773-782`, `playos_storage.h:81-90` cited |
| Sprint 12 aligned | `Sprint-12.md` carries single-identity + data-driven-sandbox foundations and the deferral note |
| Link integrity | `mdbook build` passes |
| No implementation drift | No storage/init/shell code changes are produced by this sprint |

---

## Acceptance Criteria

- [ ] The assessment states a clear feasibility verdict with a plus/minus table
- [ ] Approach A (path-scoped profiles) is recommended over Approach B (uid-per-profile)
- [ ] The "multiple local users" interpretation is recorded as console-style local profiles, not multiple interactive Linux users
- [ ] The `/data/profiles/<pid>/...` layout is documented, with game installs shared and network/log config system-wide
- [ ] The additive `PLAYOS_PROFILE_ID` + profile-scoped path env design is documented with citations
- [ ] The legacy `/data/saves/<game-id>` → default-profile migration is recorded
- [ ] The `ideas.md` wording conflict and the "profiles" overload are disambiguated
- [ ] Sprint 12 is aligned as the isolation foundation (single uid + data-driven sandbox + deferral note)
- [ ] `SUMMARY.md`, `post-mvp.md`, and `sprints/roadmap.md` are updated
- [ ] `mdbook build` passes

---

## Handoff to Post-MVP

After this sprint:

- The "many isolated local users" question has a written answer: **path-scoped console profiles**, layered on Sprint 12.
- Sprint 12 is the on-record isolation foundation, and Sprint 21 is the profile-dimension follow-up that reuses it.
- The never-planned "multiple interactive Linux users" boundary is preserved, and the top-level `/data/profiles/` vs game-internal `profiles/` collision is resolved.
- PIN/parental-controls/uid-per-profile are explicitly deferred, not scheduled.

---

## Exit Gate

The assessment is written, indexed, and link-verified; it concludes that console-style local profiles are feasible post-MVP via **path-scoped profiles** (single `playos-game` uid + `PLAYOS_PROFILE_ID` + profile-scoped Landlock), it aligns Sprint 12 as the isolation foundation, and it leaves all profile implementation unplanned.

*Previous: [Sprint 20](Sprint-20.md)*
