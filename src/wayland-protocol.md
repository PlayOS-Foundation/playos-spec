# PlayOS Private Wayland Protocol Specification

> **Authoritative repository:** `playos-runtime/protocols/`  
> **Protocol name:** `playos_v1`  
> **Generated with:** `wayland-scanner`  
> **Cross-references:** [architecture.md](architecture.md) §9.3, [playos-compositor-spec.md](playos-compositor-spec.md)

---

## Overview

Standard Wayland protocols are used wherever possible. The private PlayOS protocol covers **only** console presentation and lifecycle concerns that have no standard equivalent.

**In-scope for this protocol:**
- Registering the trusted shell and overlay roles
- Surface readiness signals
- Foreground/background transition notifications
- PlayOS-specific output information

**Out of scope (handled by control IPC, not Wayland):**
- Game installation and management
- Save data
- Networking
- System updates
- General process control

---

## Protocol File

Located at: `playos-runtime/protocols/playos-v1.xml`

```xml
<?xml version="1.0" encoding="UTF-8"?>
<protocol name="playos_v1">

  <copyright>
    Copyright © PlayOS contributors.
    SPDX-License-Identifier: MIT
  </copyright>

  <description summary="PlayOS private console compositor protocol">
    Private protocol between playos-compositor and trusted PlayOS clients.
    Not exposed to untrusted game clients.
  </description>

  <!-- ─────────────────────────────────────────────────────── -->
  <!-- playos_manager_v1                                        -->
  <!-- Global singleton — trusted clients bind this first      -->
  <!-- ─────────────────────────────────────────────────────── -->

  <interface name="playos_manager_v1" version="1">
    <description summary="PlayOS compositor management interface">
      Trusted clients bind this global to access PlayOS-specific compositor
      capabilities. Access is enforced by UNIX credentials (PLAYOS_TRUSTED_SHELL
      or PLAYOS_TRUSTED_OVERLAY environment variable check at connection time).
    </description>

    <!-- Role registration -->
    <request name="register_shell">
      <description summary="Identify this client as the trusted shell"/>
    </request>

    <request name="register_overlay">
      <description summary="Identify this client as the trusted overlay"/>
    </request>

    <!-- Compositor state events -->
    <event name="compositor_state_changed">
      <description summary="Compositor lifecycle state changed"/>
      <arg name="state" type="uint" summary="playos_compositor_state enum value"/>
    </event>

    <!-- Enumerations -->
    <enum name="compositor_state">
      <entry name="shell_foreground"                        value="0"/>
      <entry name="game_starting"                           value="1"/>
      <entry name="game_foreground"                         value="2"/>
      <entry name="playos_ui_foreground_with_game_background" value="3"/>
      <entry name="terminating_game"                        value="4"/>
    </enum>

    <enum name="error">
      <entry name="role_already_taken"  value="0" summary="Another client already holds this role"/>
      <entry name="permission_denied"   value="1" summary="Client is not trusted"/>
    </enum>
  </interface>

  <!-- ─────────────────────────────────────────────────────── -->
  <!-- playos_shell_v1                                          -->
  <!-- Interface given to the registered trusted shell          -->
  <!-- ─────────────────────────────────────────────────────── -->

  <interface name="playos_shell_v1" version="1">
    <description summary="Interface for the trusted PlayOS shell"/>

    <!-- Shell → compositor -->
    <request name="set_surface">
      <description summary="Associate a wl_surface as the shell surface"/>
      <arg name="surface" type="object" interface="wl_surface"/>
    </request>

    <request name="surface_ready">
      <description summary="Shell has rendered its first frame and is ready to display"/>
    </request>

    <!-- Compositor → shell -->
    <event name="lifecycle_event">
      <description summary="Lifecycle event for the shell"/>
      <arg name="event" type="uint" summary="playos_lifecycle_event enum value"/>
    </event>

    <event name="game_launched">
      <description summary="A game process has been spawned"/>
      <arg name="game_id" type="string"/>
    </event>

    <event name="game_exited">
      <description summary="The active game exited or crashed"/>
      <arg name="game_id"   type="string"/>
      <arg name="exit_code" type="int"/>
      <arg name="crashed"   type="uint" summary="1 if abnormal exit"/>
    </event>

    <enum name="playos_lifecycle_event">
      <entry name="foreground" value="1"/>
      <entry name="background" value="2"/>
      <entry name="suspend"    value="3"/>
      <entry name="resume"     value="4"/>
      <entry name="terminate"  value="5"/>
    </enum>
  </interface>

  <!-- ─────────────────────────────────────────────────────── -->
  <!-- playos_overlay_v1                                        -->
  <!-- Interface given to the registered trusted overlay        -->
  <!-- ─────────────────────────────────────────────────────── -->

  <interface name="playos_overlay_v1" version="1">
    <description summary="Interface for the trusted PlayOS overlay"/>

    <!-- Overlay → compositor -->
    <request name="set_surface">
      <description summary="Associate a wl_surface as the overlay surface"/>
      <arg name="surface" type="object" interface="wl_surface"/>
    </request>

    <request name="surface_ready">
      <description summary="Overlay has rendered and is ready to display"/>
    </request>

    <request name="request_dismiss">
      <description summary="Overlay requests to be hidden (e.g. user pressed Resume)"/>
    </request>

    <!-- Compositor → overlay -->
    <event name="about_to_show">
      <description summary="Compositor is about to map the overlay; overlay should prepare its frame"/>
    </event>

    <event name="about_to_hide">
      <description summary="Compositor is about to unmap the overlay"/>
    </event>

    <event name="output_info">
      <description summary="Current output dimensions for overlay layout"/>
      <arg name="width"        type="int"/>
      <arg name="height"       type="int"/>
      <arg name="refresh_mhz" type="uint" summary="Refresh rate in mHz, e.g. 60000 = 60Hz"/>
      <arg name="scale_100"   type="uint" summary="Scale factor × 100, e.g. 100 = 1.0×"/>
    </event>
  </interface>

</protocol>
```

---

## Client Trust Model

The compositor enforces trust at connection time, not through Wayland protocol negotiation:

| Client type | Trust check | Interfaces available |
|---|---|---|
| Trusted shell | `PLAYOS_TRUSTED_SHELL=1` in env | `playos_manager_v1`, `playos_shell_v1`, standard Wayland |
| Trusted overlay | `PLAYOS_TRUSTED_OVERLAY=1` in env | `playos_manager_v1`, `playos_overlay_v1`, standard Wayland |
| Active game | No trust flag | Standard Wayland only (`wl_compositor`, `wl_seat`, `xdg_wm_base`) |

The compositor does **not** advertise `playos_manager_v1` in the global registry. Trusted clients must explicitly bind by name. Untrusted clients that attempt to bind privileged interfaces receive a `wl_display.error` and are disconnected.

---

## Standard Protocols Exposed to All Clients

```
wl_compositor          surface creation
wl_shm                 shared memory buffers
wl_seat                keyboard, pointer, touch input
xdg_wm_base            toplevel and popup surfaces
wp_presentation        presentation timing (optional, for frame pacing)
```

---

## Standard Protocols Withheld from Games

```
zwp_linux_dmabuf_v1    (games use EGL/Wayland via Raylib, not raw dmabuf)
wlr_output_management  (display configuration is compositor-only)
wlr_layer_shell        (layer surfaces are compositor-policy territory)
wlr_screencopy         (no screen capture by games)
wp_drm_lease           (no DRM access by games)
```

---

## Build Integration

```makefile
# In playos-runtime/Makefile or CMakeLists.txt:
wayland-scanner client-header protocols/playos-v1.xml > gen/playos-v1-client.h
wayland-scanner server-header protocols/playos-v1.xml > gen/playos-v1-server.h
wayland-scanner private-code  protocols/playos-v1.xml > gen/playos-v1.c
```

Both `playos-compositor` (server-side) and trusted client libraries (client-side) link against the generated code.

---

## Versioning

Protocol interfaces are versioned with the `version` attribute in the XML. Bumping a version requires:
1. Adding a new `<event>` or `<request>` (never removing existing ones)
2. Incrementing `version` in the `<interface>` element
3. Updating the version bind check in compositor and client code
4. Adding a CHANGELOG entry to `playos-runtime`
