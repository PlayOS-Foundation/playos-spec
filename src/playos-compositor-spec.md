# `playos-compositor` Specification

> **Repository:** `playos-compositor`  
> **Role:** wlroots-based Wayland compositor; permanent DRM/KMS owner  
> **Language:** C  
> **Cross-references:** [architecture.md](architecture.md) §7.2, §8–9, [wayland-protocol.md](wayland-protocol.md), [sprints/Sprint-2.md](sprints/Sprint-2.md), [sprints/Sprint-4.md](sprints/Sprint-4.md)

---

## Responsibilities

`playos-compositor` owns **display policy and input routing**. It does not spawn processes, manage storage, or install games.

| Owns | Does NOT own |
|---|---|
| DRM/KMS devices and output state | Process spawning or supervision |
| wlroots backend, renderer, allocator, scene | Game installation or save management |
| Wayland display socket | Storage layout |
| Display selection, orientation, refresh, hotplug | Boot policy |
| Surface roles, z-order, visibility, focus | Network policy |
| Trusted shell and overlay identity | System updates |
| Expected game identity and first-frame activation | |
| Reserved system input interception | |
| Input routing between shell, game, overlay | |
| Direct-scanout policy | |
| Lifecycle state transitions | |
| Crash recovery (surface cleanup, return to shell) | |

---

## Initialization Order

```
main()
    │
    ├── Parse args and environment
    ├── Open log sink
    ├── wl_display_create()
    ├── wlr_backend_autocreate() or wlr_drm_backend_create()
    ├── GPU discovery (enumerate DRM devices, select by PCI vendor)
    ├── wlr_renderer_autocreate() — GBM/EGL/GLES
    ├── wlr_allocator_autocreate()
    ├── wlr_compositor_create()
    ├── wlr_output_layout_create()
    ├── xdg_wm_base setup
    ├── wlr_seat_create()
    ├── libinput backend setup (reserved key interception)
    ├── PlayOS private protocol setup (playos_manager_v1, playos_shell_v1, playos_overlay_v1)
    ├── wl_display_add_socket_auto() → "playos-0"
    ├── wlr_backend_start()
    ├── Write readiness token to PLAYOS_COMPOSITOR_READY_FD
    │
    └── wl_display_run() — event loop
```

---

## GPU Discovery

```c
/* Do not assume /dev/dri/card0 */
drmDevice *devices[MAX_DRM_DEVICES];
int count = drmGetDevices2(0, devices, MAX_DRM_DEVICES);

for (int i = 0; i < count; i++) {
    if (!(devices[i]->available_nodes & (1 << DRM_NODE_PRIMARY))) continue;
    // Resolve PCI vendor ID
    // Check if a connector is active on this device
    // Validate renderer init
    // Select if AMD (0x1002) or Intel (0x8086) with active display
}
```

**Selection priority:**
1. Device with an active connected display
2. AMD (primary platform)
3. Intel
4. First valid DRM device

Log: selected GPU path, PCI ID, connector name, preferred mode.

---

## Lifecycle State Machine

The state machine is the central PlayOS contract. **Must be explicitly tested.**

```
SHELL_FOREGROUND
    │  launch accepted (SetExpectedGame received via compositor socket)
    ▼
GAME_STARTING
    │  game commits first valid wl_buffer (first-frame rule)
    ▼
GAME_FOREGROUND ◄──────────────────────────┐
    │  PLAYOS_BUTTON_SYSTEM intercepted    │ Resume requested
    ▼                                       │
PLAYOS_UI_FOREGROUND_WITH_GAME_BACKGROUND ──┘
    │  Quit requested
    ▼
TERMINATING_GAME
    │  GameExited received from playos-init
    ▼
SHELL_FOREGROUND ◄── game exits or crashes from any state
```

State is stored as a single enum in the compositor. All state transitions are logged.

### State-specific behavior

| State | Shell surface | Game surface | Overlay surface | Input routing |
|---|---|---|---|---|
| `SHELL_FOREGROUND` | Visible, focused | Hidden | Hidden | → Shell |
| `GAME_STARTING` | Visible (launching UI) | Hidden | Hidden | → Shell |
| `GAME_FOREGROUND` | Hidden | Visible, focused | Hidden | → Game (filtered) |
| `PLAYOS_UI_...` | Hidden | Visible but unfocused | Visible, focused | → Overlay |
| `TERMINATING_GAME` | Hidden | Fading out | Visible or hidden | → Overlay |

---

## Surface Policy

### Shell surface
- Always created at startup
- Fullscreen, z-order: bottom
- Visible in `SHELL_FOREGROUND` and `GAME_STARTING`
- Background-hidden (not unmapped) in `GAME_FOREGROUND` — the surface remains alive

### Game surface
- Created when a game Wayland client maps a surface
- Only accepted if the client's `PLAYOS_LAUNCH_TOKEN` matches `expected_launch_token`
- Becomes foreground only after the first committed, non-null `wl_buffer` (first-frame rule)
- On game exit/crash: surface is destroyed or ignored; compositor transitions to `SHELL_FOREGROUND`

### Overlay surface
- Pre-spawned at startup, hidden
- Mapped above the game surface in `PLAYOS_UI_...` state
- z-order: above everything
- Receives a semi-transparent dimming layer below it (rendered by compositor)

### Scene z-order (bottom to top)
```
1. active game surface (or hidden when shell is foreground)
2. optional dimming layer (compositor-rendered, semi-transparent black)
3. overlay surface (trusted Wayland client)
4. notifications / cursor (future)
```

---

## Input Policy

### Reserved keys
`PLAYOS_BUTTON_SYSTEM` is mapped to a specific evdev key code (ROG Ally: Armory Crate button). The compositor intercepts this at the seat level via libinput; it is **never forwarded** to any Wayland client.

```c
// In handle_key event:
if (key_code == PLAYOS_SYSTEM_KEY_CODE) {
    handle_system_action();
    return;  // do NOT pass to wlr_seat_keyboard_notify_key()
}
```

### Input routing
```
libinput event
    │
    ▼ compositor seat handler
    ├── reserved key (SYSTEM, QUICK_MENU) → handle_system_action()
    ├── state == PLAYOS_UI_... → send to overlay client
    ├── state == GAME_FOREGROUND → send to game client
    └── otherwise → send to shell client
```

---

## Direct Scanout

When the game's fullscreen surface buffer is compatible with the DRM output plane:
1. Check buffer format, modifier, and size against the plane's supported formats
2. If compatible: assign buffer directly to DRM plane (skip GPU composition)
3. If overlay is visible or compatibility check fails: fall back to GPU composition

**Scanout is an optimization, not an MVP correctness requirement.** All transitions log whether scanout or composition was used.

---

## Output Management

- Enumerate all outputs on DRM device initialization
- Select the primary output (built-in display) by connector type (eDP or DSI preferred)
- Set preferred mode (native resolution and refresh rate)
- Handle `wlr_output.events.destroy` for hotplug removal — log and continue
- External display support is post-MVP

---

## Private Protocol Implementation (server side)

See [wayland-protocol.md](wayland-protocol.md) for the full XML.

The compositor implements the **server side** of:
- `playos_manager_v1` — global, binds trusted client roles
- `playos_shell_v1` — emits lifecycle events and game state to the shell
- `playos_overlay_v1` — notifies overlay of show/hide; receives dismiss requests

---

## Crash Recovery Invariants

A game crash must **never**:
- Leave the display black for more than 500ms
- Reveal a Linux terminal or TTY
- Require a compositor restart

When the game Wayland client disconnects unexpectedly:
1. Compositor destroys the stale game surface
2. Transitions immediately to `SHELL_FOREGROUND`
3. Shell surface is made visible and focused
4. `GameExited(crashed=true)` is relayed to trusted clients

---

## Build

```cmake
# playos-compositor/CMakeLists.txt
find_package(wlroots REQUIRED)
find_package(wayland-server REQUIRED)
find_package(libdrm REQUIRED)

target_sources(playos-compositor PRIVATE
    src/main.c
    src/compositor.c
    src/output.c
    src/input.c
    src/seat.c
    src/scene.c
    src/protocols/playos-v1.c     # generated by wayland-scanner
)
target_link_libraries(playos-compositor wlroots::wlroots wayland-server drm)
```
