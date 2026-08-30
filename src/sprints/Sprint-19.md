# Sprint 19 — Marketplace Assessment (Post-MVP)

**Goal:** Produce an initial evaluation, analysis, and assessment of the `playos-marketplace` repository — what exists today, what the specs and sprints already say about a store, and how the marketplace will complement PlayOS — without writing any marketplace implementation.

**Primary Outcome:** A decision-ready `Sprint-19.md` record that (a) maps the empty `playos-marketplace` stub against the existing spec/sprint backlog, (b) identifies the spec gaps and naming discrepancies that block implementation, and (c) proposes a spec-first sequencing so the marketplace can later be built consistently with PlayOS.

**Status:** 🟡 Post-MVP — assessment only; not scheduled. No marketplace code is implemented in this sprint.

**Prerequisites:** MVP stable (Sprint 15–16); Wi-Fi (`playos-net`, Sprint 16) is the network prerequisite for any on-device store downloads; the SDK story (`playos-sdk`, post-MVP) is the publisher prerequisite.

---

## Why This Sprint Exists

`playos-marketplace` exists as a repository but contains **no implementation** — only a `README.md`, an `AGENTS.md`, a `.github/copilot-instructions.md`, and an issue template. Its own guidance says marketplace behaviour "is specified in `playos-spec` (Part XI)", and its issue template references "Part X — Package Format" and "Part XI — Cloud and Marketplace". Those spec parts **do not exist**. The word "marketplace" appears nowhere in `playos-spec`.

Meanwhile `playos-spec` *does* contain scattered store/package intent — "Store Integration and Download Manager", "Signed `.play` Content Packages", and a historical `.play` package note — but no marketplace chapter, no package-format chapter, no catalog model, and no entitlement model. The package extension also disagrees: the marketplace stub says `.gpk`, the spec says `.play`.

This sprint turns that disorganised state into a single assessment: what the marketplace should be, how it complements PlayOS, and what must be specified first.

---

## Assessment Inputs

- **`playos-marketplace` is a stub.** Files present: `README.md`, `AGENTS.md`, `.github/copilot-instructions.md`, `.github/ISSUE_TEMPLATE/implementation-task.md`. No source, services, or package-format definitions.
- **Marketplace golden rules** (from its `AGENTS.md`):
  1. **SDK-first** — a developer can publish without owning a PlayOS device.
  2. **Multiple store sources** — official, community, OEM, private, and LAN; never hard-code a single store.
  3. **Self-hostable** — a store can be run by communities, OEMs, or individuals.
  4. **Spec-first** — package format (`.gpk`), signing, entitlements, and catalog behaviour are specified before implementation.
  5. **Trust** — verify package signatures; respect permissions and the trust model.
- **Spec references to a store/package (no "marketplace" by name):**
  - [`post-mvp.md`](../post-mvp.md) Tier 3 — **Store Integration and Download Manager** (API addition `playos_store.h`; depends on Wi-Fi, signed `.play` packages, cloud saves) and **Signed `.play` Content Packages** (signed archive: `manifest.json`, binary, assets, content hash tree; replaces plain directory installs).
  - [`roadmap.md`](roadmap.md) post-MVP list — "Download manager and store integration" and "Signed `.play` content packages".
  - [`ideas.md`](../ideas.md) §14.2 — historical "Later `.play` package" requirements: deterministic metadata, signature verification, integrity hashes, atomic installation, versioned migrations, strict save-data separation.
  - [`Sprint-12`](Sprint-12.md) — "Store-level signing and distribution" explicitly out of scope for security hardening.
  - [`Sprint-15`](Sprint-15.md) — "Store integration and SDK signing/distribution" deferred to post-MVP.
  - [`Sprint-16`](Sprint-16.md) — store downloads listed as a Tier-1 post-MVP consumer of Wi-Fi.
  - [`architecture.md`](../architecture.md) §14 — "Wi-Fi, Bluetooth, SSH, cloud saves | Post-MVP" (store implied via cloud saves).
- **Existing manifest schema:** [`schemas/game-manifest-v1.json`](../../schemas/game-manifest-v1.json) — the current per-game `manifest.json` contract that any package format must extend or reference.
- **Naming discrepancy:** `playos-marketplace` uses **`.gpk`**; `post-mvp.md`, `ideas.md`, and `roadmap.md` use **`.play`**.

---

## Assessment

### What the marketplace is

Per its own docs, the PlayOS Marketplace is the open platform for **publishing, discovering, installing, and updating** PlayOS applications, games, themes, and developer content. Its design constraints are already well stated and are worth keeping:

- **SDK-first** publishing (no device required to publish).
- **Multiple store sources** (official, community, OEM, private, LAN) — a client is store-agnostic.
- **Self-hostable** stores.
- **Spec-first** package format, signing, entitlements, and catalog.
- **Trust** via signature verification and permission respect.

### How it complements PlayOS

Today the MVP is a **local launcher**: games are installed by dropping directories into `/data/games/<game-id>/` with a `manifest.json`, and the only "distribution" mechanism is the offline `.playosb` system-update bundle. There is no way to discover, download, verify, install, or update *content* on-device, and no way for a third party to publish without hand-delivering files.

The marketplace closes that loop and turns PlayOS from a launcher into a platform:

```
publisher ──SDK──▶ build .gpk ──▶ publish ──▶ catalog (official/community/OEM/private/LAN)
                                                      │
player ──Wi-Fi──▶ discover ──▶ download ──▶ verify ──▶ install ──▶ run/update ──▶ revoke
```

Specifically it:

1. **Completes the delivery path** that `post-mvp.md` already names ("Store Integration and Download Manager"). Wi-Fi (Sprint 16) is the transport; the marketplace is the source and policy layer.
2. **Enables third-party content** without manual file transfer — the natural partner of the `playos-sdk` (publishers) and the shell's game library (players).
3. **Gives content a secure, atomic lifecycle** — signature verification + content hash tree + atomic install + rollback, reusing the patterns already proven by Sprint 11's A/B update engine and Sprint 12's manifest signing.
4. **Adds entitlements and revocation** — ownership/licensing that the MVP's plain-directory model cannot express.
5. **Enables themes and developer content**, not just games, matching the marketplace's broader mandate and future shell theming.

### Mapping to existing architecture

| Marketplace concern | Existing PlayOS foundation to build on |
|---|---|
| On-device download | Wi-Fi (`playos-net`, Sprint 16) |
| Package verification | Signed manifests (Sprint 12), HMAC/Hash verification (Sprint 11) |
| Atomic install / rollback | A/B update engine pattern (Sprint 11), `/data/downloads` staging |
| Storage | `/data/games/<id>/`, `/data/downloads/`, `/data/updates/` (Sprint 6) |
| Client surface | `playos_store.h` (named in `post-mvp.md`), shell "Store" screen, `playos-tools` CLI |
| Publishing | `playos-sdk` (post-MVP) — SDK-first publishing with no device |
| Trust | `security-model.md` trust zones; `manifest.json` + signatures |

### Spec gaps and open decisions (the real blockers)

1. **No "Part X — Package Format" and no "Part XI — Cloud and Marketplace" exist.** The marketplace's own rule is *spec-first*, so this is the bottleneck: the spec must be authored before marketplace code.
2. **Package extension naming is inconsistent.** Marketplace says `.gpk`; spec says `.play`. **Recommendation:** adopt **`.gpk`** as the canonical content-package extension (it covers games, themes, and developer content, and is already the marketplace repo's term), then reconcile the `.play` references in `post-mvp.md`, `ideas.md`, and `roadmap.md`. This is a decision to lock during spec authoring, not silently here.
3. **No catalog model.** Nothing specifies a signed catalog format, discovery protocol, or how multiple store sources (official/community/OEM/private/LAN) are configured and selected by a client.
4. **No entitlement model.** Ownership, free-vs-paid, device limits, offline entitlements, and revocation are unspecified. **Recommendation:** v1 should be **free content with signed, device-local entitlements — no payments, no billing, no DRM.** Commerce is explicitly out of scope for the core platform.
5. **Client integration surface is only a name.** `playos_store.h` is referenced but not specified; the shell has no "Store" screen in any sprint; `playos-tools` currently covers system updates only.

### Verdict

The marketplace is a **natural and necessary post-MVP complement** — it is the content-economy layer that the MVP deliberately deferred, and its repository's golden rules are a sound, coherent philosophy. It is **not** a code problem yet: it is a **spec problem**. The highest-value next step is to author the missing spec parts (package format + marketplace/cloud), resolve the `.gpk`/`.play` naming, and define a minimal catalog + entitlement + client surface — *then* implement the marketplace repo.

---

## Recommended Spec-First Sequencing

1. **Part X — Package Format (`.gpk`).** Define the signed package: `manifest.json` + binary + assets + content hash tree + signature, atomic install, versioned migrations, save-data separation. This absorbs and supersedes the current "Signed `.play` Content Packages" post-MVP item.
2. **Part XI — Cloud and Marketplace.** Define catalog format + discovery, store-source selection (official/community/OEM/private/LAN), entitlements, and revocation — free-content-only v1, no payments.
3. **Client surface.** Specify `playos_store.h` (query, download, install progress, entitlement check), a shell "Store" screen, and a `playos-tools`/SDK `publish` command.
4. **Then implement `playos-marketplace`** (catalog service, publishing flow, client integration) against those specs.

---

## Scope

### In Scope (this sprint)

- This `Sprint-19.md` document.
- The assessment, gap analysis, and recommended sequencing above.
- Cross-linking from `SUMMARY.md` and `post-mvp.md`.

### Explicitly Out of Scope / Not Planned Now

- Any marketplace service, client, or publishing implementation.
- The actual "Part X / Part XI" spec chapters (future work).
- Payments, billing, DRM, or storefront UI polish.
- Changing the `.play` references in `post-mvp.md`/`ideas.md`/`roadmap.md` now — the naming reconciliation is a decision for Part X authoring.

---

## Required Repository Changes

| Repo | Required work |
|---|---|
| `playos-spec` | Add `Sprint-19.md`; link from `SUMMARY.md` and `post-mvp.md` |
| *(none else)* | No implementation repositories change in this sprint |

---

## Expected Files and Directories

```text
playos-spec/src/sprints/Sprint-19.md   # NEW: this assessment
playos-spec/src/SUMMARY.md             # UPDATE: link
playos-spec/src/post-mvp.md            # UPDATE: entry
```

---

## Agent Task Breakdown

### Task Status Grid

| Task ID | Task | Primary repo | Status | Notes / evidence |
|---|---|---|---|---|
| S19-T1 | Survey store/package references and reconcile with marketplace `AGENTS.md` | `playos-spec` | not started | `post-mvp.md`, `roadmap.md`, `ideas.md`, Sprints 12/15/16 |
| S19-T2 | Assess package-format gap and `.gpk` vs `.play` naming | `playos-spec` | not started | `schemas/game-manifest-v1.json`, `ideas.md` §14.2 |
| S19-T3 | Assess catalog + entitlement + trust + multi-store model | `playos-spec` | not started | `security-model.md`, marketplace golden rules |
| S19-T4 | Define spec-first sequencing + client integration surface | `playos-spec` | not started | `playos_store.h`, shell Store screen, SDK publish |

### S19-T1 — Survey store/package references

- Enumerate every store/package mention in `playos-spec`: `post-mvp.md` Tier 3, `roadmap.md` post-MVP list, `ideas.md` §14.2 and §22, and the "out of scope" notes in Sprint 12, Sprint 15, and Sprint 16.
- Compare them against `playos-marketplace/AGENTS.md` golden rules and confirm that the word "marketplace" is absent from the spec and that "Part X / Part XI" are dangling references.
- Record the reconciliation outcome: marketplace is spec-blocked, not code-blocked.

**Done when:** the sprint lists the complete set of store/package references and states the dangling "Part X / Part XI" gap.

### S19-T2 — Package-format gap and naming

- Confirm the current game content contract is `manifest.json` (see `schemas/game-manifest-v1.json`) with plain-directory install, and that the future signed package is only sketched in `post-mvp.md` ("Signed `.play` Content Packages") and `ideas.md` §14.2.
- Document the `.gpk` (marketplace) vs `.play` (spec) discrepancy and recommend **`.gpk`** as canonical, with `.play` references to be reconciled during Part X authoring.
- List the package properties that must be specified: manifest, binary, assets, content hash tree, signature, atomic install, versioned migrations, save-data separation.

**Done when:** the sprint names the package-format gap, the naming discrepancy, and a concrete recommendation.

### S19-T3 — Catalog, entitlement, and trust model

- Assess what is missing: signed catalog format, discovery protocol, store-source selection (official/community/OEM/private/LAN), and entitlements/revocation.
- Recommend a minimal v1: **free content, signed catalogs, device-local entitlements, no payments/billing/DRM**.
- Map trust to `security-model.md`: package signature verification before install, content hash tree, and entitlement checks that respect the untrusted-game boundary.

**Done when:** the sprint records a minimal catalog + entitlement + trust model and explicitly excludes payments/DRM.

### S19-T4 — Spec-first sequencing and client surface

- Define the four-step sequencing: Part X package format → Part XI marketplace/cloud → client surface → implementation.
- Specify the client surface to be defined later: `playos_store.h` (query, download, install progress, entitlement), a shell "Store" screen, and a `playos-tools`/SDK `publish` command.
- State that no marketplace code is written until those specs exist (honouring the repo's own spec-first rule).

**Done when:** the sprint records a spec-first sequence and the concrete client surface to be specified next.

---

## Verification and Evidence

| Evidence | How it is produced |
|---|---|
| Assessment recorded | `Sprint-19.md` present with gap analysis, verdict, and sequencing |
| Roadmap indexed | `SUMMARY.md` and `post-mvp.md` link the sprint |
| Naming discrepancy documented | `.gpk` vs `.play` explicitly recorded with a recommendation |
| Link integrity | `mdbook build` passes |
| No implementation drift | No marketplace code or spec chapters are produced by this sprint |

---

## Acceptance Criteria

- [ ] The assessment states what `playos-marketplace` is, what exists today, and that it is an empty stub
- [ ] All existing store/package references in `playos-spec` are enumerated
- [ ] The dangling "Part X / Part XI" spec references are identified as the primary blocker
- [ ] The `.gpk` vs `.play` naming discrepancy is documented with a recommendation
- [ ] A minimal catalog + entitlement + trust model is proposed (free-content-only v1, no payments/DRM)
- [ ] A spec-first sequencing is defined (package format → marketplace/cloud → client surface → implementation)
- [ ] A marketplace implementation is explicitly left unplanned until those specs exist
- [ ] `SUMMARY.md` and `post-mvp.md` are updated
- [ ] `mdbook build` passes

---

## Handoff to Post-MVP

After this sprint:

- The marketplace has a written assessment and a clear "spec-first" prerequisite list.
- The next authoring step (Part X package format, then Part XI marketplace/cloud) can begin without re-deriving the gap analysis.
- The `.gpk` naming decision is flagged for Part X authoring to lock.

---

## Exit Gate

The assessment is written, indexed, and link-verified; it clearly concludes that `playos-marketplace` is a necessary post-MVP complement that is currently **spec-blocked** (missing Part X/Part XI, unresolved `.gpk` vs `.play`, undefined catalog/entitlements), and it defines a spec-first sequencing so implementation can proceed only after the contracts exist.

*Previous: [Sprint 17](Sprint-17.md) | Next: [Sprint 20](Sprint-20.md)*
