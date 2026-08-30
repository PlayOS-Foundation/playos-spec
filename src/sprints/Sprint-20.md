# Sprint 20 — Native Media & Browser Client Strategy (Post-MVP)

**Goal:** Produce a written plus/minus and feasibility assessment for a **controller-first native media and browser client** strategy — Spotify, YouTube, YouTube Music, and a lightweight browser — launched from `playos-shell` as first-class PlayOS apps. Netflix is explicitly out of scope. This sprint records the assessment and scopes a bounded de-risking spike; no media client is integrated.

**Primary Outcome:** A decision-ready `Sprint-20.md` record that (a) replaces the earlier Chromium/CEF framing with a native-client recommendation, (b) analyses the launch, compositor, audio, GPU, toolchain, and security consequences, (c) states a clear feasibility verdict, and (d) scopes an optional bounded host-only de-risking spike.

**Status:** 🟡 Post-MVP — assessment only; not scheduled. No implementation work is approved.

**Prerequisites:** MVP stable (Sprint 15–16); wlroots compositor with Wayland session (Sprint 4/5); ALSA audio (Sprint 8); Wi-Fi (`playos-net`, Sprint 16); musl-only constraint (ADR-0003); no D-Bus/BusyBox policy (Sprint 16, `network-options.md`); security model on paper (Sprint 12).

---

## Why This Sprint Exists

PlayOS is a games-first handheld console, but a recurring product question is whether it should also run the streaming services users expect on a portable device — Spotify, YouTube, and a general browser. Netflix is the natural fourth ask, but it has no first-party Linux client and its DRM path is fundamentally incompatible with PlayOS's musl-only, ALSA-only, no-Widevine constraints. This sprint therefore splits the problem:

- **In scope:** Spotify, YouTube, YouTube Music, and a lightweight browser via **native/thin clients** that respect the existing musl, ALSA, and Wayland architecture.
- **Out of scope:** Netflix, Chromium/CEF/Electron, X11/Xwayland, and any Widevine/EME DRM.

The recommendation below is a **native-client-first** strategy, not a web-runtime strategy. It deliberately keeps PlayOS controller-first and avoids reopening the locked musl/ALSA decisions.

---

## Assessment Inputs

The assessment rests on the following authoritative facts, verified against source and spec:

- **Launch model is ELF-only.** The shell requests launch over `control.sock` IPC to `playos-init`; `playos-init` is the only fork/exec (`playos-init/src/supervisor.c`). The child path is hard-coded to `executable` with an `access(..., X_OK)` gate and `execl(exe_path, exe_path, NULL)` (`supervisor.c:731,806`). The shell manifest validator additionally requires `executable` + `architecture` and checks `access(F_OK)` (`playos-shell/src/screen_library.c:153-265`). A media/web launch target needs a new manifest `type`/`url`/`media_uri`, an init exec branch, and shell validation relaxed for that type. The manifest schema `game-manifest-v1.json` already has `additionalProperties: true`, so extending it is non-breaking.
- **Compositor exposes standard Wayland globals but does not forward input.** wlroots 0.20 creates `wlr_compositor`, `wlr_xdg_shell`, and a `wlr_seat` (`playos-compositor/src/compositor.c:286-298`), so arbitrary Wayland clients can attach and create `xdg_toplevel`s. However there is **no `wlr_seat_set_keyboard`/`pointer`/`touch`** forwarding — `src/system_button.c:63` states non-reserved keys are "intentionally not forwarded". Surfaces are forced fullscreen and there is one fixed game role. A native client therefore must be driven over **its own IPC** (e.g. `mpv --input-ipc-server`), not through the Wayland seat.
- **Audio is ALSA-only.** [ADR-0007](../adr/ADR-0007-audio-stack.md) mandates direct ALSA PCM with no PulseAudio or PipeWire in v0.1.0. `playos-platform-api` audio is mixer-only. Native clients that speak ALSA directly (mpv, librespot/spotifyd backends) fit; clients that require PulseAudio/PipeWire do not.
- **GPU is GBM/EGL/GLES2/Mesa only.** AMDGPU + Mesa `radeonsi` EGL/ES/GBM is the whole graphics story. There is **no** VAAPI/VDPAU/V4L2/dmabuf-import/hardware-video-decode path in the compositor or defconfigs; video playback would be software-decoded. This is acceptable for 1080p and below, but 4K/HDR is effectively off the table.
- **Toolchain is x86_64 + musl.** [ADR-0003](../adr/ADR-0003-libc-choice.md) and `playos_ally_defconfig` set `BR2_x86_64`, `BR2_TOOLCHAIN_BUILDROOT_MUSL`, no glibc. `mpv`, `librespot`, `spotifyd`, `WPE WebKit`, and `Cog` are musl-buildable in principle, but their Buildroot packaging and any bundled prebuilts must be validated — none currently exists.
- **Sandbox is not implemented.** `playos-init/src/supervisor.c` spawns with only `setsid()` + env + `execl()`; no seccomp/Landlock/`PR_SET_NO_NEW_PRIVS`/user namespaces yet. `security-model.md` is the design, not the code. Media clients are network-facing parsers (HLS/DASH/YT playlists, HTML/JS in WPE), so the Sprint 12 sandbox should land before they run unconfined.
- **Spec has zero existing media/browser content.** Grep across `playos` for `librespot|spotifyd|mpv|yt-dlp|ytdl|ytmusic|youtube|WPE|Cog|webkit|InnerTube|innertube` finds no existing media-client integration (only this sprint and unrelated raylib noise). Buildroot grep for `BR2_PACKAGE_(MPV|FFMPEG|PYTHON3|RUST|LIBRESPOT|SPOTIFYD|WPE|WEBKIT|COG)` finds **no non-sensitive matches** — none of these packages are present. The never-planned list explicitly includes "Browser-based shell or WebAssembly runtime", "X11 / Xwayland", and "Cloud gaming".

---

## Feasibility Assessment

### Verdict

**Viable as a post-MVP, controller-first native-client strategy — with per-service caveats.** Spotify, YouTube, and YouTube Music can be delivered as native/thin clients that speak ALSA and attach as Wayland surfaces, **without** reopening ADR-0003 or ADR-0007. The browser (WPE WebKit + Cog) is the riskiest item because of its musl-build and WebKit security surface; it should be treated as an optional spike, not a product commitment. Netflix stays out of scope.

This is a materially better fit than the earlier Chromium/CEF path because it does **not** require glibc, PulseAudio/PipeWire, or Widevine. The remaining hard work is integration plumbing and packaging, not architectural reversal.

### Recommended stack

| Service | Recommended client | Audio | DRM | Notes |
|---|---|---|---|---|
| Spotify | `librespot` (preferred) / `spotifyd` | ALSA backend | None (own streaming, no EME) | `librespot` is a Rust daemon; drive it over its IPC/CLI, not the Wayland seat |
| YouTube | `mpv` + `yt-dlp` | ALSA (`--ao=alsa`) | None for standard content | `yt-dlp` resolves streams; `mpv` plays them. Prefer a standalone/static binary on-device (see YouTube Music) |
| YouTube Music | `mpv` + `yt-dlp`/InnerTube search | ALSA | None | Music is the same pipeline as YouTube; search/library UX decides the wrapper shape |
| Browser | `WPE WebKit` + `Cog` | ALSA (WebKit audio) | **No Widevine** | Lightweight embedded WebKit; no Netflix/DRM. Highest musl/security risk |
| Netflix | **Out of scope** | — | Widevine (glibc, proprietary, L3 ~720p) | No native Linux client; needs a Chromium/CEF/Widevine path PlayOS rejects |

### YouTube Music — PlayOS app shape

YouTube Music should be a **PlayOS app**, not a separate browser tab. Two options:

- **Option A (recommended):** a thin Raylib controller-first wrapper that drives `mpv` via `--input-ipc-server` JSON and performs search via `yt-dlp`/InnerTube. The wrapper renders a PlayOS-styled list/now-playing UI in the existing Raylib backend and maps controller input to IPC commands. This reuses the shell's own UI stack and is the smallest integration.
- **Option B:** a dedicated headless backend (`innertube-rs` or `ytmusicapi`) exposing library/playlist/queue semantics, with `mpv` as the renderer. Only pursue this if library/playlist/offline-cache UX demands more than Option A.

Recommendation: start with **Option A**; escalate to Option B only when the library/queue UX is proven to need it.

### Plus / Minus

| Dimension | Plus | Minus |
|---|---|---|
| Platform reach | Spotify/YouTube/YouTube Music + a browser turn the handheld into a genuine entertainment device | Media is not the MVP's growth area; it partially competes with the spec's games-first positioning |
| Architecture fit | Native clients speak ALSA directly and attach as Wayland surfaces — no ADR-0003/ADR-0007 reversal | The compositor's single-game role and missing `wl_seat` input must be worked around (per-client IPC), and forced fullscreen means no window/tab management |
| Toolchain | `mpv`/`librespot`/`spotifyd`/`WPE` are all musl-buildable in principle | None are in Buildroot today; `WPE WebKit` on musl is a flagged risk and must be validated in the spike |
| Audio | Direct ALSA is a first-class fit | Any client that assumes PulseAudio/PipeWire must be configured or patched to ALSA |
| Security | Per-client IPC keeps the attack surface bounded and avoids trusting the Wayland seat | Network-facing parsers (HLS/DASH/HTML/JS) should not run unconfined — the Sprint 12 sandbox is a prerequisite |
| Performance | 1080p software decode is realistic on the Ally's CPU; 16 GB RAM is ample | 4K/HDR is effectively out of scope without VAAPI/VDPAU/dmabuf import; browser memory can be large |
| DRM | Spotify/YouTube standard content need no Widevine/EME | YouTube/Spotify premium and *all* of Netflix are DRM-gated and stay out of scope |
| Cost/benefit | The launch-target change (manifest `type` + init exec branch + shell validation) is small and non-breaking | Browser (WPE) and YouTube Music search UX are the real multi-sprint items; packaging five new Buildroot packages is non-trivial |

### Hard blockers

1. **Compositor input model.** There is no `wl_seat` keyboard/pointer/touch forwarding. Native clients are therefore **not interactive over the Wayland seat** — they must be driven over their own IPC (`mpv --input-ipc-server`, `librespot` control socket, etc.) with PlayOS-side controller binding. This is a design constraint, not a blocker, but it must be respected in every client.
2. **No sandbox.** `supervisor.c` has no seccomp/Landlock/`PR_SET_NO_NEW_PRIVS`. Network-facing media/browser clients should not ship unconfined; the Sprint 12 sandbox must land first.
3. **No Buildroot packages.** `mpv`, `ffmpeg`, `librespot`/`spotifyd`, `WPE WebKit`, and `Cog` (plus their Rust/Python build deps) are absent from every defconfig. Packaging and cross-compiling them against musl is the bulk of the actual work.
4. **No hardware video decode.** No VAAPI/VDPAU/dmabuf import means software decode only; 4K/HDR is effectively off the table. This bounds the feature to 1080p-and-below streaming.
5. **Netflix/DRM is a hard non-starter.** Widevine is proprietary, glibc-only, Google-licensed, and L3 on generic Linux x86_64 (~720p ceiling). It contradicts ADR-0003 and requires a distribution agreement. Netflix is therefore **out of scope**, not "deferred".

### Integration design (how it fits the existing lifecycle)

The existing game lifecycle is reused almost unchanged:

1. **Manifest:** extend `game-manifest-v1.json` with a `type` field (`native` | `media` | `web`) plus `url`/`media_uri`. The schema already has `additionalProperties: true`, so this is non-breaking.
2. **Shell:** relax the validator at `playos-shell/src/screen_library.c:153-265` for `media`/`web` types — do not require `executable`+`architecture` when a `url`/`media_uri` is present, and render a media/web card in the library.
3. **Init:** branch the exec path in `playos-init/src/supervisor.c` (currently ELF-only at `:731` X_OK gate, `:806` `execl`) so `media`/`web` types spawn the appropriate client (`mpv`, `librespot`, `Cog`) with a fixed argv derived from the manifest.
4. **Input:** do **not** build compositor `wl_seat` forwarding for this sprint. Instead, each client is driven over its own IPC, and the controller mapping lives in the thin wrapper (or, for YouTube Music Option A, in the Raylib wrapper itself).
5. **Audio:** clients use ALSA directly (`mpv --ao=alsa`, `librespot`/`spotifyd` ALSA backend), so the Tier 2 "Dedicated Audio Service" is **not** a prerequisite.
6. **Security:** gated on the Sprint 12 sandbox before any device integration.

### What it would actually take

In dependency order:

1. **Land the Sprint 12 sandbox** (seccomp + Landlock + `PR_SET_NO_NEW_PRIVS`) so media/browser clients are not unconfined.
2. **Package and cross-compile** `ffmpeg` + `mpv`, `librespot`/`spotifyd`, and (optionally) `WPE WebKit` + `Cog` for `x86_64-musl` in Buildroot.
3. **Extend the launch lifecycle** (manifest `type`/`url`/`media_uri`, shell validator relaxation, init exec branch) as a non-breaking change.
4. **Build the controller-first wrappers** (Spotify control, YouTube/YouTube Music search+play UI) driving each client over its own IPC.
5. **Validate the Wayland attach** for `mpv` (`--vo=gpu --gpu-context=wayland` or `wlshm`) and `Cog` under the compositor's forced-fullscreen, single-game-role model.

Only after those would "spawn a media/web client as a `type: media|web` launch target" be a small, well-understood change.

---

## Recommended De-Risking Spike (Bounded, Host-Only)

If ever funded, a single bounded spike would, in order:

1. **Package/build probe** — confirm `mpv`/`ffmpeg`/`librespot`/`spotifyd`/`WPE WebKit`/`Cog` actually cross-compile against `x86_64-musl` in Buildroot, and record which are non-starters. **No defconfig merge yet.**
2. **Wayland attach probe** — validate `mpv` and `Cog` attach as fullscreen Wayland surfaces under a host nested compositor, using `--vo=gpu --gpu-context=wayland`/`wlshm`. Confirm the game-role fit.
3. **IPC/controller probe** — confirm `mpv --input-ipc-server` and `librespot`/`spotifyd` can be driven without the Wayland seat, and sketch the controller→IPC mapping.
4. **YouTube Music UX spike** — prototype Option A (thin Raylib wrapper + mpv IPC + yt-dlp/InnerTube search) on a host to prove search/play/now-playing is achievable.
5. **Explicit stop** — do **not** proceed to device integration, Buildroot merge, sandbox implementation, or Netflix.

---

## Scope

### In Scope (this sprint)

- This `Sprint-20.md` document.
- The feasibility analysis, recommended stack, plus/minus, blocker list, integration design, and verdict above.
- Definition of the bounded host-only de-risking spike (S20-T1…T4) — *not* its execution.

### Explicitly Out of Scope / Not Planned

- Any `mpv`/`ffmpeg`/`librespot`/`spotifyd`/`WPE`/`Cog` integration or Buildroot packaging.
- Netflix, Chromium/CEF/Electron, Widevine/EME, X11/Xwayland.
- Compositor `wl_seat` input forwarding, audio service, sandbox, or hardware-video-decode implementation.
- Changing the existing shell, compositor, or init in this sprint.

---

## Required Repository Changes

| Repo | Required work |
|---|---|
| `playos-spec` | Replace `Sprint-20.md` with this native-client strategy; update `SUMMARY.md`, `post-mvp.md`, and `roadmap.md` |
| *(none else)* | No implementation repositories change in this sprint |

---

## Expected Files and Directories

```text
playos-spec/src/sprints/Sprint-20.md   # REPLACE: this assessment
playos-spec/src/SUMMARY.md             # UPDATE: link title
playos-spec/src/post-mvp.md            # UPDATE: entry
playos-spec/src/sprints/roadmap.md     # UPDATE: sprint-plan row
```

---

## Agent Task Breakdown

### Task Status Grid

| Task ID | Task | Primary repo | Status | Notes / evidence |
|---|---|---|---|---|
| S20-T1 | Native-client feasibility (musl build, ALSA, Wayland attach) | `playos-spec` | not started | ADR-0003, ADR-0007, `compositor.c`, no Buildroot packages |
| S20-T2 | Integration design (manifest `type`, init exec branch, shell validation, IPC) | `playos-spec` | not started | `supervisor.c:731,806`, `screen_library.c:153-265`, manifest schema |
| S20-T3 | YouTube Music app shape (Option A vs B) + browser risk | `playos-spec` | not started | mpv IPC, yt-dlp/InnerTube, WPE/Cog musl risk |
| S20-T4 | Bounded host-only de-risking spike definition | `playos-spec` | not started | build probe, Wayland probe, IPC probe, YT Music UX spike |

### S20-T1 — Native-client feasibility

- Confirm `mpv`/`ffmpeg`, `librespot`/`spotifyd`, and `WPE WebKit`/`Cog` are musl-buildable in principle and speak ALSA directly, tying the conclusion to [ADR-0003](../adr/ADR-0003-libc-choice.md) and [ADR-0007](../adr/ADR-0007-audio-stack.md).
- Record that **none** of these packages exist in any Buildroot defconfig (grep evidence), and flag `WPE WebKit`-on-musl as the highest risk.
- State this as a packaging/spike gate, not a code task.

**Done when:** the sprint records the recommended stack and identifies the exact spike step (S20-T4 step 1) that would validate musl buildability.

### S20-T2 — Integration design

- Confirm the launch path is ELF-only (`playos-init/src/supervisor.c:731,806`) and the shell validator requires `executable`+`architecture` (`playos-shell/src/screen_library.c:153-265`).
- Confirm the manifest schema has `additionalProperties: true`, so a `type`/`url`/`media_uri` extension is non-breaking.
- Confirm the compositor forwards no `wl_seat` input (`playos-compositor/src/system_button.c:63`), so clients must be driven over their own IPC rather than the Wayland seat.

**Done when:** the sprint records the non-breaking manifest/init/shell changes and the per-client IPC input workaround with source citations.

### S20-T3 — YouTube Music app shape + browser risk

- Recommend Option A (thin Raylib wrapper driving `mpv` via `--input-ipc-server` + yt-dlp/InnerTube search) over Option B (dedicated headless backend).
- Record the browser (WPE/Cog) as the riskiest item: musl build risk, no Widevine, and WebKit security surface requiring the Sprint 12 sandbox.

**Done when:** the sprint states the YouTube Music recommendation and marks the browser as an optional spike rather than a product commitment.

### S20-T4 — Bounded host-only de-risking spike definition

- Define the five-step spike: build probe → Wayland attach probe → IPC/controller probe → YouTube Music UX spike → explicit stop.
- Scope it strictly to **host-only**, with **no** Buildroot merge, **no** device integration, **no** sandbox implementation, and **no** Netflix.
- State the acceptance that marks the spike complete and the condition under which it would *not* advance to device integration.

**Done when:** the sprint records a bounded, stoppable spike definition and a default recommendation to keep Netflix out of scope.

---

## Verification and Evidence

| Evidence | How it is produced |
|---|---|
| Assessment recorded | `Sprint-20.md` present with recommended stack, plus/minus, blockers, integration design, and verdict |
| Roadmap indexed | `SUMMARY.md`, `post-mvp.md`, and `roadmap.md` link the sprint with the new title |
| Grounded analysis | Each blocker cites a source file, ADR, or grep result |
| Link integrity | `mdbook build` passes |
| No implementation drift | No media/browser files, Buildroot changes, or compositor/audio changes are produced by this sprint |

---

## Acceptance Criteria

- [ ] The assessment states a clear feasibility verdict with a plus/minus table
- [ ] A recommended native-client stack (Spotify `librespot`, YouTube/YouTube Music `mpv`+`yt-dlp`, browser `WPE`/`Cog`) is recorded
- [ ] Netflix is explicitly documented as out of scope (Widevine + glibc + Google license)
- [ ] YouTube Music is assessed as a PlayOS app with Option A recommended over Option B
- [ ] The non-breaking integration design (manifest `type`/`url`/`media_uri`, init exec branch, shell validation, per-client IPC) is documented with citations
- [ ] The missing compositor `wl_seat` input forwarding is recorded as a design constraint, not a blocker, with the IPC workaround
- [ ] The Sprint 12 sandbox and missing Buildroot packages are recorded as prerequisites
- [ ] A bounded host-only de-risking spike is defined with an explicit stop
- [ ] `SUMMARY.md`, `post-mvp.md`, and `roadmap.md` are updated
- [ ] `mdbook build` passes

---

## Handoff to Post-MVP

After this sprint:

- The "Spotify/YouTube/YouTube Music/browser?" question has a written answer and a recommended native-client strategy.
- Netflix remains fully out of scope, and Chromium/CEF/Electron/X11/Xwayland remain never-planned.
- The locked decisions (musl-only, ALSA-only) are preserved, not reopened.
- A future spike, if funded, can pick up S20-T4's bounded scope without re-deriving the assessment.

---

## Exit Gate

The assessment is written, indexed, and link-verified; it concludes that a controller-first native media/browser client strategy (Spotify `librespot`, YouTube/YouTube Music `mpv`+`yt-dlp`, browser `WPE WebKit`+`Cog`) is viable as a post-MVP direction **without** reopening the musl/ALSA ADRs, while Netflix is explicitly out of scope — and it scopes an optional bounded host-only spike while leaving all device integration unplanned.

*Previous: [Sprint 19](Sprint-19.md) | Next: [Sprint 21](Sprint-21.md)*
