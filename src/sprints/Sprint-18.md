# Sprint 18 — C# Shell Reimplementation Assessment (Post-MVP Spike)

**Goal:** Produce a written plus/minus and feasibility assessment of re-implementing `playos-shell` in C#, and record it as a post-MVP investigation sprint. No C# shell is implemented in this sprint.

**Primary Outcome:** A decision-ready `Sprint-18.md` record that (a) analyses the runtime, packaging, interop, and rendering consequences of a C# shell, (b) states a clear feasibility verdict, and (c) scopes an optional bounded host-only de-risking spike — while explicitly leaving a product-direction C# rewrite unplanned.

**Status:** 🟡 Post-MVP — assessment only; not scheduled. No implementation work is approved.

**Prerequisites:** MVP stable (Sprint 15–16); the Raylib 6.0 shell landed (Sprint 5.5); `rcore_playos.c` is the single rendering backend (ADR-0006); the musl-only constraint is in force (ADR-0003).

---

## Why This Sprint Exists

The shell is currently ~8k lines of C plus a vendored Raylib with a custom backend. A recurring question is whether a managed language (C# / .NET) would reduce the memory-safety and manual-parsing risk of that C code and speed UI iteration. This sprint does **not** commit to a rewrite; it answers the question with an assessment, so the roadmap can explicitly accept or reject the direction.

This is a *decision-support* sprint, not an implementation sprint. Its output is the assessment itself, and the default recommendation is to **not** pursue a C# shell as a product direction.

---

## Assessment Inputs

The assessment rests on the following authoritative facts:

- [`ADR-0003 — libc Choice (musl)`](../adr/ADR-0003-libc-choice.md) — musl only, no glibc. This is the decisive runtime constraint.
- [`ADR-0006 — UI Framework (Raylib)`](../adr/ADR-0006-ui-framework.md) — Raylib is the single UI framework for shell, overlay, and games.
- [`architecture.md`](../architecture.md) §14 explicitly excludes "libc other than musl" from PlayOS v1.
- [`playos-shell-spec.md`](../playos-shell-spec.md) — shell responsibilities; Raylib is rendering-only, and controller input is read directly from evdev (`src/input.c`) so reserved SYSTEM/QUICK_MENU buttons survive.
- The shell links `wayland-client`, `wayland-egl`, EGL, GLESv2, `libplayos` (from `playos-platform-api`), vendored static `raylib`, and optionally `libplayos-trusted` (from `playos-runtime`).
- Wayland protocol code is generated to C from `playos-v1.xml` + xdg-shell via `wayland-scanner`.

---

## Assessment Constraints (Locked)

These constraints are not re-negotiated by this sprint; any C# rewrite would have to re-establish them:

- **musl-only is non-negotiable** ([ADR-0003](../adr/ADR-0003-libc-choice.md), [architecture.md](../architecture.md) §14).
- **Single rendering framework** ([ADR-0006](../adr/ADR-0006-ui-framework.md)). A rewrite that forks a second rendering path must justify it, not silently drop it.
- **Shell invariants carried by the existing C code** and required of any rewrite:
  - always alive, supervised by `playos-init`;
  - 60 fps target, controller-only navigation;
  - no blocking I/O on the render thread;
  - no direct IPC socket access except the trusted evdev input path;
  - rendering stops while a game is foreground, but the process stays alive.

---

## Feasibility Assessment

### Verdict

Technically feasible as a **host-side proof-of-concept**; **not advisable as a shippable on-device product direction**. The decisive factors are: (1) .NET-on-musl / NativeAOT risk, (2) Buildroot toolchain effort, and (3) loss of the single Raylib backend story.

### Plus / Minus

| Dimension | Plus | Minus |
|---|---|---|
| Memory safety | Eliminates buffer overflows and manual string-parsing bugs in the current C shell (e.g. hand-rolled JSON in `main.c`) | GC/allocator behaviour and working-set size on an always-on 60 fps embedded process must be re-validated |
| UI iteration | Richer abstractions (records, LINQ, test framework) can speed state/UI iteration | Raylib's UI layer is intentionally thin; C# does not remove the need to bind Raylib or re-derive rendering |
| Native interop | P/Invoke + source generators can wrap a C ABI | The entire surface — `libplayos`, `wayland-client`/`wayland-egl`, EGL, GLESv2, trusted IPC, evdev — must be wrapped or generated; marshaling on musl is untested |
| Runtime & packaging | Self-contained .NET removes a host dependency | Supported Linux RIDs assume glibc; `linux-musl-x64` exists but NativeAOT still depends on the OS libc/ICU and needs validation; Buildroot has no first-class .NET SDK package |
| Rendering | — | Raylib's native library is C with a custom `rcore_playos.c` backend; C# bindings (`Raylib-cs`) target upstream Raylib, so the PlayOS backend is lost or must be kept in C and P/Invoked — weakening the ADR-0006 single-backend rationale |
| Wayland protocols | Community C# Wayland bindings exist | `playos-v1.xml` + xdg-shell are generated to C via `wayland-scanner`; a C# binding generator would need the private protocols ported |
| Total cost/benefit | Modest long-term maintainability upside | Replaces ~8k lines of working, shipped C with a high-risk multi-week-to-month effort for a persistent controller UI that is not the product's growth area |

### Runtime (musl / NativeAOT) — hard blocker to validate

- .NET's supported Linux runtime identifiers assume glibc (`linux-x64`). Alpine's `linux-musl-x64` RID exists, but self-contained deployment still relies on the OS libc, and ICU/globalization behaviour historically differs on musl.
- NativeAOT reduces startup and footprint but does **not** remove the libc/ICU dependency; threading, P/Invoke marshaling, and finalizer behaviour on musl are exactly the areas with the least field coverage.
- **This is the single highest-risk item and must be proven by a spike, not assumed.** Until a self-contained NativeAOT binary runs on the actual musl rootfs with the same EGL/Wayland bindings, a C# shell is a research bet, not a plan.

> All runtime claims above are architectural and **to be validated by a spike**; they are not asserted as tested.

### Buildroot packaging — hard blocker to estimate

- Buildroot has no first-class .NET SDK package. Shipping means either a host .NET SDK toolchain with cross-compilation, or NativeAOT produced on a host and injected into the image.
- Either path is new `br2-external` machinery with no precedent in this repository — substantial, unproven infrastructure work.

### Native interop surface — medium

A C# shell would need bindings for, at minimum:

- `libplayos` (the public C ABI): lifecycle, storage paths, device info, logical input, audio/display/power queries, structured logging.
- `libwayland-client`, `libwayland-egl`, EGL, GLESv2.
- Trusted IPC: the `playos-runtime` restricted control client (`control.sock`) and optional `libplayos-trusted`.
- Direct evdev input, to preserve reserved SYSTEM/QUICK_MENU button survival exactly as `src/input.c` does today.

### Raylib backend loss — strongest architectural minus

- Raylib is C. The PlayOS value is the custom `rcore_playos.c` backend shared by shell, overlay, and games.
- A C# shell either P/Invokes a C Raylib build (keeping the backend in C, so C# gains little in the rendering hot path) or re-derives Wayland/EGL/GLES3 surface management in managed code — a second rendering path that directly contradicts ADR-0006's unified-framework rationale.
- Neither option reduces the amount of C the project must own; both add a managed/native boundary across every frame and every input event.

### What *is* cheap and worth doing

- A host-only C# proof-of-concept of the **state and screen layer only**: screen enum + navigation stack, `manifest.json` parsing, power/thermal model types, toast/screenshot state. This exercises the "C# is nicer for UI state" hypothesis without touching Buildroot, musl, Wayland, or Raylib.

---

## Recommended De-Risking Spike (Bounded)

If ever funded, a single bounded spike would, in order:

1. **Host-only state/screen POC** — a .NET console/unit-test harness against a re-typed `manifest.json` and state model.
2. **musl proof** — compile a minimal self-contained NativeAOT "hello" binary and run it on the existing musl rootfs (no Wayland/EGL), recording libc/ICU/startup/size results.
3. **Binding probe** — generate a C# binding for `playos-v1.xml` + xdg-shell and complete one `wl_surface` round-trip on a host Wayland compositor.
4. **Explicit stop** — do **not** proceed to a device C# shell, Buildroot packaging, or a Raylib-backend replacement.

---

## Scope

### In Scope (this sprint)

- This `Sprint-18.md` document.
- The feasibility analysis and verdict above.
- Definition of the bounded de-risking spike (S18-T1…T4) — *not* its execution.

### Explicitly Out of Scope / Not Planned

- Any C# shell implementation.
- Buildroot / .NET toolchain work.
- Raylib backend port or replacement.
- Changing the existing C shell.

---

## Required Repository Changes

| Repo | Required work |
|---|---|
| `playos-spec` | Add `Sprint-18.md`; link from `SUMMARY.md` and `post-mvp.md` |
| *(none else)* | No implementation repositories change in this sprint |

---

## Expected Files and Directories

```text
playos-spec/src/sprints/Sprint-18.md   # NEW: this assessment
playos-spec/src/SUMMARY.md             # UPDATE: link
playos-spec/src/post-mvp.md            # UPDATE: entry
```

---

## Agent Task Breakdown

### Task Status Grid

| Task ID | Task | Primary repo | Status | Notes / evidence |
|---|---|---|---|---|
| S18-T1 | Runtime + Buildroot feasibility (musl / NativeAOT) | `playos-spec` | not started | ADR-0003, architecture §14 |
| S18-T2 | Native interop + Wayland protocol bindings | `playos-spec` | not started | `libplayos`, wayland-client, `playos-v1.xml` |
| S18-T3 | Raylib backend loss + rendering-path options | `playos-spec` | not started | ADR-0006, `rcore_playos.c` |
| S18-T4 | Bounded host-only de-risking spike definition | `playos-spec` | not started | state/screen-layer POC only |

### S18-T1 — Runtime + Buildroot feasibility

- Confirm the .NET supported Linux RID situation against ADR-0003 (musl only): `linux-x64` (glibc) vs `linux-musl-x64`, and NativeAOT's remaining libc/ICU dependency.
- Assess Buildroot packaging options: host .NET SDK cross-compilation vs host-produced NativeAOT injected into the image, and the new `br2-external` machinery each requires.
- Record startup time, binary size, and working-set expectations as **to be validated**, not proven.

**Done when:** the sprint records a clearly-labelled runtime/packaging risk verdict and identifies the exact spike step (S18-T4 step 2) that would validate it.

### S18-T2 — Native interop + Wayland protocol bindings

- Enumerate the concrete surface a C# shell must bind: `libplayos` C ABI, `libwayland-client`/`libwayland-egl`, EGL, GLESv2, trusted IPC (`control.sock`), and direct evdev.
- Assess how `playos-v1.xml` + xdg-shell (currently generated to C by `wayland-scanner`) would be generated for C#, and whether community C# Wayland bindings can absorb the private protocols.
- Identify marshaling/threading risks specific to musl.

**Done when:** the sprint records the full binding surface and names the binding-probe step (S18-T4 step 3) that would de-risk it.

### S18-T3 — Raylib backend loss + rendering-path options

- Evaluate the two options: P/Invoke a C Raylib build that keeps `rcore_playos.c`, versus re-deriving Wayland/EGL/GLES3 in managed code.
- State why the second option violates ADR-0006's single-framework rationale, and why the first leaves the rendering hot path in C.
- Conclude with the recommendation that this is the strongest architectural minus.

**Done when:** the sprint names the Raylib-backend loss as the decisive architectural minus and ties it to ADR-0006.

### S18-T4 — Bounded host-only de-risking spike definition

- Define the four-step spike: host state/screen POC → musl "hello" proof → binding probe → explicit stop.
- Scope it strictly to **host-only**, with **no** Buildroot work and **no** Raylib backend replacement.
- State the acceptance that marks the spike complete and the condition under which it would *not* advance to a product C# shell.

**Done when:** the sprint records a bounded, stoppable spike definition and a default recommendation not to pursue a C# rewrite.

---

## Verification and Evidence

| Evidence | How it is produced |
|---|---|
| Assessment recorded | `Sprint-18.md` present with plus/minus table and verdict |
| Roadmap indexed | `SUMMARY.md` and `post-mvp.md` link the sprint |
| Link integrity | `mdbook build` passes |
| No implementation drift | No C# files or Buildroot changes are produced by this sprint |

---

## Acceptance Criteria

- [ ] The assessment states a clear feasibility verdict with a plus/minus table
- [ ] The musl/NativeAOT and Buildroot risks are labelled "to be validated", not asserted as proven
- [ ] The Raylib backend loss is identified as the strongest architectural minus and tied to ADR-0006
- [ ] A bounded host-only de-risking spike is defined with an explicit stop
- [ ] A C# rewrite is explicitly left unplanned as a product direction
- [ ] `SUMMARY.md` and `post-mvp.md` are updated
- [ ] `mdbook build` passes

---

## Handoff to Post-MVP

After this sprint:

- The "C# shell?" question has a written answer and a default recommendation (do not pursue as a product direction).
- A future spike, if funded, can pick up S18-T4's bounded scope without re-deriving the assessment.

---

## Exit Gate

The assessment is written, indexed, and link-verified; it clearly concludes that a C# shell reimplementation is technically feasible as a host POC but not advisable as a product direction, and it scopes an optional bounded spike while leaving the actual rewrite unplanned.

*Previous: [Sprint 17](Sprint-17.md)*
