# Sprint 2 — Compositor Skeleton and Wayland Session

**Goal:** Build a minimal `playos-compositor` on wlroots that creates a Wayland session, exposes only the minimum public protocols required for a single fullscreen client, and renders a test client in QEMU headless mode and nested Wayland mode.

**Primary Outcome:** `playos-compositor` starts under `playos-init`, creates a Wayland socket, signals readiness, accepts one fullscreen client, and presents a visible frame through the wlroots scene/output pipeline.

**Prerequisites:** Sprint 1 complete — `playos-init` runs as PID 1, supervision works, and the trusted control IPC is available.

---

## Why This Sprint Exists

Sprint 2 proves the graphics session model before any ROG Ally-specific DRM work:

1. The compositor exists as a supervised process, not just an idea in the spec.
2. A stable Wayland socket and lifecycle exist before the shell is implemented.
3. Headless and nested workflows exist so later graphics sprints can iterate quickly.

---

## Start Condition Checklist

- Sprint 1 QEMU boot still works.
- `playos-init` can supervise a child process reliably.
- `playos-runtime\protocols\playos-v1.xml` exists and may be extended.
- A Linux host or CI environment exists for wlroots builds.

---

## Decisions Locked for This Sprint

- **Language:** C99 for the compositor.
- **Build system:** CMake.
- **Compositor base:** wlroots.
- **Primary test modes:** headless for CI/QEMU and nested Wayland for developer validation.
- **Surface policy:** one visible fullscreen surface only.
- **Privileged protocol scope:** skeleton only. Do not implement launch, overlay, or game lifecycle semantics here yet.

---

## Scope

### In Scope

- wlroots startup and shutdown
- Wayland socket creation
- scene graph and one-output render loop
- `xdg_wm_base` support for one fullscreen toplevel
- compositor readiness signal to `playos-init`
- skeleton private PlayOS Wayland protocol XML
- test client
- Buildroot package integration

### Explicitly Out of Scope

- native DRM/KMS on physical hardware
- first-frame foreground switching logic
- trusted overlay UI
- real shell UX
- direct scanout optimisation

---

## Required Repository Changes

| Repo | Required work |
|---|---|
| `playos-compositor` | Implement the wlroots compositor skeleton and test client |
| `playos-runtime` | Maintain the private protocol XML and scanner-generated glue |
| `playos-refdistro` | Add Buildroot packaging and dependencies for wlroots and the compositor |
| `playos-spec` | Clarify protocol or backend strategy if implementation exposes gaps |

---

## Expected Files and Directories

### `playos-compositor`

```text
CMakeLists.txt
include/
└── compositor.h

src/
├── main.c
├── compositor.c
├── backend.c
├── output.c
├── scene.c
├── xdg_shell.c
├── seat.c
├── trusted_client.c
└── readiness.c

tests/
├── headless/
└── nested/

tools/
└── test-client/
    ├── CMakeLists.txt
    └── src/main.c
```

### `playos-runtime`

```text
protocols/
└── playos-v1.xml
```

### `playos-refdistro`

```text
br2-external/package/playos-compositor/
├── Config.in
└── playos-compositor.mk
```

---

## Agent Task Breakdown

### Task Status Grid

Update the **Status** column as work progresses: `not started` → `in progress` → `blocked` or `done`.

| Task ID | Task | Primary repo | Status | Notes / evidence |
|---|---|---|---|---|
| S2-T1 | Bootstrap the compositor project | `playos-compositor` | not started | |
| S2-T2 | Create backend selection and startup flow | `playos-compositor` | not started | |
| S2-T3 | Implement the minimal renderable session | `playos-compositor` | not started | |
| S2-T4 | Implement trusted-shell identity skeleton | `playos-compositor` | not started | |
| S2-T5 | Add the private Wayland protocol skeleton | `playos-runtime`, `playos-compositor` | not started | |
| S2-T6 | Add a test client | `playos-compositor` | not started | |
| S2-T7 | Wire `playos-init` supervision and readiness | `playos-refdistro`, `playos-compositor` | not started | |
| S2-T8 | Integrate with Buildroot and tests | `playos-refdistro`, `playos-compositor` | not started | |

### S2-T1 — Bootstrap the compositor project

- Create the buildable `playos-compositor` project.
- Add wlroots, wayland-server, wayland-protocols, pixman, libxkbcommon, and libdrm headers as dependencies.
- Define a central `struct playos_compositor`.

**Done when:** a host build produces `playos-compositor`.

### S2-T2 — Create backend selection and startup flow

- Support `PLAYOS_BACKEND=headless` and `PLAYOS_BACKEND=wayland`.
- Default to headless in CI/QEMU.
- Initialize `wl_display`, event loop, backend, renderer, allocator, scene graph, and output layout.
- Create the Wayland socket and store its name predictably.

**Done when:** startup logs show successful backend creation and socket creation.

### S2-T3 — Implement the minimal renderable session

- Support `xdg_wm_base`.
- Accept one `xdg_toplevel` surface.
- Force fullscreen layout for the active client.
- Present frames through the normal wlroots frame event loop.

**Done when:** a client surface can be mapped, rendered, and unmapped cleanly.

### S2-T4 — Implement trusted-shell identity skeleton

- Define a temporary trusted-shell registration mechanism for this sprint.
- The recommended flow is:
  1. compositor starts
  2. `playos-init` launches the intended shell/test client with a documented environment variable
  3. compositor records that client as trusted once connected

- Do not rely on this temporary identity mechanism beyond Sprint 6/7 without replacement.

**Done when:** the compositor can distinguish the first trusted client from an untrusted client in logs/state.

### S2-T5 — Add the private Wayland protocol skeleton

- Keep `playos-v1.xml` intentionally small in this sprint.
- Generate scanner output in the compositor build.
- Implement only the minimum needed to prove protocol wiring, such as shell registration and a placeholder lifecycle event.

**Done when:** the scanner-generated code builds and the compositor can advertise the private global.

### S2-T6 — Add a test client

- Build a minimal raw Wayland or simple graphics client.
- Connect to the compositor socket.
- Create one `xdg_toplevel`.
- Render a visible frame with PlayOS identification text or colour.
- Exit cleanly when closed.

**Done when:** the client can be used as the sprint acceptance target in both headless and nested modes.

### S2-T7 — Wire `playos-init` supervision and readiness

- Replace the compositor stub from Sprint 1 with the real compositor binary.
- Add a readiness signal from the compositor to PID 1.
- PID 1 must not start the trusted client until the compositor is ready.

**Done when:** boot logs show a supervised compositor start followed by trusted client start only after readiness.

### S2-T8 — Integrate with Buildroot and tests

- Add the Buildroot package metadata.
- Ensure the QEMU image includes the compositor binary and required libraries.
- Add headless integration tests and a nested manual validation path.

**Done when:** the image boots and reaches a visible compositor-driven frame path in QEMU/nested environments.

---

## Protocol and Readiness Guidance

### Wayland socket

- Default socket name: `playos-0`
- The compositor owns socket creation.
- PID 1 passes `WAYLAND_DISPLAY=playos-0` to child clients after readiness.

### Readiness signalling

Use one explicit mechanism only. Recommended options:

1. a pipe inherited from `playos-init`
2. a readiness file in `/run/playos/`
3. a one-shot status message over the trusted control IPC

Whichever option is chosen, document it and keep it for Sprint 3 unless it proves insufficient.

---

## Implementation Guidance

### Rendering path

- Headless mode may use wlroots' headless backend and software/pixman rendering if needed.
- Nested mode should validate the same scene logic using a nested Wayland backend.
- Keep all surface ownership in compositor state. Do not spread `wlr_*` globals across multiple files without structure.

### Input for this sprint

- Minimal seat support is enough.
- Virtual keyboard/pointer devices are acceptable.
- Do not begin ROG Ally input mapping here; that belongs in Sprint 3.

### Logging

- Write compositor logs under `/run/playos/log/compositor.log`.
- Log backend choice, socket name, output creation, client map/unmap, and readiness.

---

## Verification and Evidence

| Evidence | How it is produced |
|---|---|
| Socket proof | log or runtime listing showing the Wayland socket exists |
| Render proof | headless screenshot/pixel read or nested visual confirmation |
| Supervision proof | `playos-init` log showing compositor start after boot |
| Readiness proof | timestamped readiness marker before trusted client launch |
| Protocol proof | build output showing `wayland-scanner` generated files are compiled |

---

## Acceptance Criteria

- [ ] `playos-compositor` builds cleanly against wlroots
- [ ] the compositor starts under `playos-init`
- [ ] a Wayland socket is created and passed to child clients
- [ ] the compositor signals readiness before trusted client launch
- [ ] a test client connects and maps one fullscreen surface
- [ ] the rendered frame is observable in headless or nested validation
- [ ] the compositor can run in nested Wayland mode for developer iteration
- [ ] the private `playos-v1.xml` skeleton is generated with `wayland-scanner`
- [ ] Buildroot packages and image integration work end-to-end
- [ ] QEMU headless validation remains automated

---

## Handoff to Sprint 3

Sprint 3 may assume:

- a functioning compositor binary already exists
- `playos-init` can supervise and wait for compositor readiness
- a stable Wayland session bootstrap exists in QEMU/dev environments
- the private protocol XML is available and build-integrated

Sprint 3 must focus on physical hardware bring-up, not rebuild the software session model from scratch.

---

## Exit Gate

`playos-compositor` starts under `playos-init`, creates a Wayland session, signals readiness, and renders a fullscreen test client in headless QEMU and nested developer mode.

*Previous: [Sprint 1](Sprint-1.md) | Next: [Sprint 3](Sprint-3.md)*
