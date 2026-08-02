# Sprint 2 — Compositor Skeleton and Wayland Session

**Goal:** Build a minimal `playos-compositor` on wlroots that creates a Wayland session, accepts one trusted fullscreen client, and presents it on screen (headless in QEMU; nested Wayland for developer iteration).

**Primary Outcome:** `playos-compositor` starts, creates a Wayland socket, and a test client connects, maps a fullscreen surface with a colored frame, and the compositor presents it correctly.

**Prerequisites:** Sprint 1 complete — `playos-init` running as PID 1 and control IPC working.

---

## Key Deliverables

### `playos-compositor` — wlroots Compositor

Create the `playos-compositor` repository with a CMake or Meson build. Implement in C.

**Backend selection (in priority order for this sprint):**

| Mode | When used |
|---|---|
| DRM/KMS | Physical hardware (Sprint 4+) |
| Headless | QEMU CI — outputs render to memory only |
| Nested Wayland | Developer desktops — compositor runs inside an existing Wayland session |

Use wlroots' backend auto-detection or a `PLAYOS_BACKEND` environment variable to select.

**Core wlroots initialization:**
- Initialize wlroots display and event loop
- Create backend (headless or nested for this sprint)
- Initialize renderer (GLES or pixman for headless)
- Initialize allocator
- Initialize output layout and create one output
- Create Wayland socket (default: `playos-0`)
- Set `WAYLAND_DISPLAY=playos-0` in child environment

**Surface and scene management:**
- Initialize wlroots scene graph
- Support `xdg_wm_base` — accept one `xdg_toplevel` surface
- Fullscreen policy: all clients are treated as fullscreen; no floating windows
- One foreground surface at a time (shell surface initially)
- Render loop: handle output frame events, render scene, present

**Trusted shell identity (skeleton):**
- Accept one connection with a special environment variable: `PLAYOS_TRUSTED_SHELL=1`
- Mark that client as the trusted shell in compositor state
- All other clients are treated as potential games (untrusted role)

**Input (minimal):**
- Initialize wlroots seat
- For headless/nested: create virtual keyboard and pointer (no real device required)
- Route all input to the current foreground surface

**Wayland protocols exposed this sprint:**
- `wl_compositor`, `wl_shm`, `wl_seat`, `xdg_wm_base`
- Do NOT expose privileged protocols yet

### Private PlayOS Wayland Protocol — Skeleton

Define the protocol XML in `playos-runtime/protocols/playos-v1.xml`:

```xml
<!-- Skeleton — full definition in Sprint 7 -->
<protocol name="playos_v1">
  <interface name="playos_shell_v1" version="1">
    <!-- Identify this client as the trusted shell -->
    <request name="register_as_shell"/>
    <!-- Emitted when a lifecycle event occurs -->
    <event name="lifecycle_event">
      <arg name="event" type="uint"/>
    </event>
  </interface>
</protocol>
```

Generate C/C++ bindings with `wayland-scanner`. This is the extension point; full content comes in Sprint 7.

### Test Client

A minimal Raylib or raw Wayland test client that:
- Connects to `playos-0`
- Creates an `xdg_toplevel` surface
- Paints the surface with a solid color
- Draws the PlayOS name and sprint number as text
- Exits cleanly on `xdg_toplevel.close`

This client is the acceptance test target and the predecessor to `playos-shell`.

### `playos-init` Integration

- `playos-init` now spawns the real `playos-compositor` binary instead of a stub
- Pass `WAYLAND_DISPLAY` and `PLAYOS_TRUSTED_SHELL=1` to `playos-shell` after compositor is ready
- Compositor readiness signal: compositor writes a readiness token to a socket or pipe that `playos-init` monitors

### Buildroot Integration
- Add `package/playos-compositor/` to the `br2-external` tree
- Add `wlroots` and its dependencies (wayland, wayland-protocols, libxkbcommon, pixman, libdrm for headers) to the build

---

## Acceptance Criteria

- [ ] `playos-compositor` compiles cleanly against wlroots in the Buildroot environment
- [ ] Compositor starts, creates a Wayland socket, and `WAYLAND_DISPLAY` is set
- [ ] Test client connects, maps a fullscreen surface, and compositor presents it
- [ ] Colored frame from test client is visible in headless output (verified via screenshot or pixel read)
- [ ] Compositor runs nested in a developer's Wayland session for iteration
- [ ] `playos-init` starts the compositor and waits for readiness before continuing
- [ ] Compositor restart by `playos-init` works when it exits unexpectedly
- [ ] Protocol XML skeleton is generated and compiled with `wayland-scanner`
- [ ] `make qemu-build && make qemu-run` still works end-to-end
- [ ] CI passes

---

## Repositories Primarily Involved

| Repo | Work |
|---|---|
| `playos-compositor` | wlroots compositor implementation |
| `playos-runtime` | Private Wayland protocol XML skeleton, `wayland-scanner` build rules |
| `playos-refdistro` | Buildroot packages for `playos-compositor`, `wlroots` and deps |
| `playos-spec` | ADR on wlroots version pin and backend strategy |

---

## Testing Approach

- Unit tests: wlroots event loop startup, socket creation, client connection/disconnection
- QEMU headless: compositor starts, test client connects, pixel output verified programmatically
- Nested Wayland: developer-only manual test for visual validation
- CI runs headless path only

---

## Exit Gate

`playos-compositor` starts under `playos-init`, creates a Wayland session, and renders a test client's fullscreen surface in headless QEMU.

*Previous: [Sprint 1](Sprint-1.md) | Next: [Sprint 3](Sprint-3.md)*
