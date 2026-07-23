---
rfc: 0012
title: Engine Integration Contract
status: Draft
authors: []
created: 2026-07-22
---

# RFC-0012: Engine Integration Contract

> **Status:** Draft

## Summary

This RFC formalises the **engine-agnostic boundary** between the PlayOS
Platform API and game engines.  It defines the precise contract that
engines and the Platform API must follow to coexist in one application,
the architecture of input bridging, the rendering boundary, and the
requirements for engine integration kits.  It is the normative
foundation for Part VII (Engine Integration) of the PlayOS Book.

## Motivation

PlayOS is engine-agnostic in principle (RFC-0001, Principle 2), but the
principle alone does not answer practical questions:

- How does a game get input from both its engine and the Platform API
  without conflicts?
- Who owns the window, the rendering context, and the frame buffer?
- How does the overlay work when the engine controls rendering?
- What makes an engine "integrated" vs. "compatible"?

Without a formal contract, each engine integration would make different
assumptions, fragmenting the developer experience and risking
engine-specific behaviour leaking into the Platform API.

## Guide-Level Explanation

Think of the engine and the Platform API as two libraries your game
links against.  They live in the same process but don't talk to each
other:

```text
Your Game
    ├── Engine (rendering, physics, audio)    ← your choice
    └── PlayOS Platform API (input, storage)  ← the platform
```

- The Engine owns the window, the rendering context, and every pixel
  on screen.
- The Platform API owns input (logical buttons), save data, lifecycle,
  and platform services.
- Neither includes the other's headers.
- Neither manages the other's lifecycle.

The Platform API doesn't know or care whether you're using Raylib,
SDL, Godot, or a custom engine.  It just works.

## Reference-Level Explanation

### 1. Separation of Concerns

| Concern | Owner | Notes |
|---|---|---|
| Window creation | Engine | `InitWindow()`, `SDL_CreateWindow()`, etc. |
| Rendering context | Engine | OpenGL, Vulkan, DirectX, Metal |
| Frame buffer / swap chain | Engine | Engine manages presentation |
| Scene graph / ECS | Engine | Game logic |
| Physics simulation | Engine | Collision, forces |
| Audio playback | Engine | Sound effects, music mixing |
| Asset loading | Engine | Textures, models, sounds |
| Button input (logical) | Platform API | `Input::Pressed(Button::A)` |
| Axis input (analog) | Platform API | `Input::GetAxis(Axis::LeftX)` |
| Save data paths | Platform API | `Storage::GetSavePath()` |
| Capability discovery | Platform API | `Capabilities::Has(...)` |
| Lifecycle | Platform API | `Lifecycle::Init/Update/Shutdown` |
| Overlay | Platform API | Activated by runtime, not app |
| Cloud services | Platform API | Saves, leaderboards, achievements |
| Display info | Platform API | `Display::Current()` for safe area, refresh rate |
| Permission checks | Platform API | `Permissions::GetState(...)` |

### 2. Coexistence Rules

#### Rule 1: Separate headers

The Platform API MUST NOT `#include` any engine header.  An engine MUST
NOT `#include` any Platform API header except through the application's
own code.

```cpp
// ✅ Correct
#include "raylib.h"         // Engine
#include "playos/playos.h"  // Platform API

// ❌ Wrong — Platform API must not include engine headers
// Inside playos/display.h:
// #include "raylib.h"  // NEVER
```

#### Rule 2: Separate loops, same frame

The engine's render loop and `Lifecycle::Update()` coexist in the same
`while (running)` block.  `Update()` is called once per frame, before
or after rendering.

```cpp
int main() {
    InitWindow(1280, 720, "My Game");  // Engine
    Lifecycle::Init();                  // Platform API

    while (!WindowShouldClose()) {
        Lifecycle::Update();            // Poll input, advance platform

        // Read input via Platform API
        if (Input::Pressed(Button::A)) { /* ... */ }

        BeginDrawing();                 // Engine
        ClearBackground(BLACK);
        DrawText("Hello", 10, 10, 20, WHITE);
        EndDrawing();
    }

    Lifecycle::Shutdown();
    CloseWindow();
    return 0;
}
```

#### Rule 3: No type sharing

The Platform API MUST NOT use engine types (`Vector2`, `Texture2D`,
`Color`, `SDL_Rect`).  The engine MUST NOT use Platform API types
(`Button`, `Axis`, `Capability`).  If conversion is needed, it happens
in the integration kit, not in the Platform API.

#### Rule 4: Input independence

The engine MAY have its own input system (keyboard, mouse, raw
controller).  The Platform API provides a parallel, logical input layer
(`Button::A`, `Axis::LeftX`).  Both can coexist:

```cpp
// Engine input — raw keyboard
if (IsKeyDown(KEY_SPACE)) { player.Jump(); }

// Platform API input — logical controller button
if (Input::Pressed(Button::A)) { player.Jump(); }
```

The application chooses which to use.  For console compliance, logical
input (Platform API) is preferred.

#### Rule 5: Lifecycle ownership

The application's `main()` owns both the engine window and
`Lifecycle::Init/Shutdown`.  Neither the engine nor the Platform API
manages the other's lifecycle.  The engine MUST NOT call
`Lifecycle::Init()` or `Lifecycle::Shutdown()`.

### 3. Input Bridging Architecture

Input bridging translates engine input events into Platform API logical
buttons.  It is implemented in the **input backend**, not in the
application.

```text
Physical device (evdev, XInput, HID)
    ↓
Platform API input backend (engine-specific)
    ↓  reads engine input state
    ↓  translates to logical buttons
    ↓
Input::Pressed(Button::A)  ← application calls this
```

#### Integration kit responsibility

Each engine integration kit MUST provide an input backend that maps the
engine's gamepad API to PlayOS logical buttons:

| Engine | Gamepad API | Backend |
|---|---|---|
| Raylib | `IsGamepadButtonPressed()` | `RaylibInputBackend` (exists) |
| SDL | `SDL_GameControllerGetButton()` | `SDLInputBackend` (planned) |
| Godot | `InputEventJoypadButton` | `GodotInputBackend` (future) |

The backend is selected at CMake configure time:

```cmake
# In the application's CMakeLists.txt:
set(PLAYOS_INPUT_BACKEND "raylib")  # or "sdl", "evdev", "xinput"
add_subdirectory(../playos-platform-api playos-api)
```

#### Required button mapping

Every input backend MUST map at minimum:

| PlayOS Button | Required |
|---|---|
| `A` (confirm) | ✅ |
| `B` (back) | ✅ |
| `X` | ✅ |
| `Y` | ✅ |
| `DPadUp` | ✅ |
| `DPadDown` | ✅ |
| `DPadLeft` | ✅ |
| `DPadRight` | ✅ |
| `Home` | ✅ |
| `QuickSettings` | ✅ |
| `LeftShoulder` | Recommended |
| `RightShoulder` | Recommended |
| `LeftStick` + `RightStick` axes | Recommended |

The device profile defines the physical-to-logical mapping; the input
backend implements it.  See RFC-0006 (Device Profile Format).

### 4. Rendering Boundary

The rendering boundary is the division between engine-owned pixels and
platform-owned display information.

#### Engine owns

- Window creation and destruction.
- Rendering context (OpenGL, Vulkan, etc.).
- Frame buffer and swap chain.
- All pixels inside the application surface.

#### Platform API provides

- `Display::Current()` — physical width, height, refresh rate.
- `Display::SafeArea()` — insets for notches, rounded corners, bezels.
- `Display::Orientation()` — current device orientation.

The engine SHOULD use `Display::Current()` to configure its window and
viewport.  The engine SHOULD respect `Display::SafeArea()` for critical
UI elements (HUD, menus, subtitles).

#### Overlay rendering

The overlay is rendered by the compositor, above the engine's frame
buffer.  The engine does not draw into the overlay surface.  When the
overlay is active:

1. The compositor captures the engine's last frame.
2. The compositor renders the overlay (Shell UI) on top.
3. The engine is paused (`Lifecycle::Update()` is not called).

The engine does not need to handle overlay rendering at all.

### 5. Integration Kit Requirements

An engine integration kit MUST provide:

| Component | Description |
|---|---|
| **CMake integration** | Snippet to link Platform API alongside engine |
| **Input backend** | Translates engine input → PlayOS logical buttons |
| **Lifecycle wiring** | Example `main()` showing dual-loop pattern |
| **Display wiring** | Example showing `Display::Current()` for window config |
| **Starter project** | Complete sample: window + input + lifecycle |
| **Documentation** | README with build instructions and API usage |

An integration kit MUST NOT:

- Modify Platform API headers.
- Add engine-specific types to the Platform API.
- Require the engine for the Platform API to compile.
- Override Platform API behaviour.

### 6. Engine Compatibility Levels

| Level | Description | Example |
|---|---|---|
| **Compatible** | Engine coexists with Platform API in same process; developer writes both engine and Platform API calls | Custom engine |
| **Integrated** | Input backend provided; dual-loop starter project available; documented | SDL (planned) |
| **Reference** | Full integration kit maintained by PlayOS project; used in Shell and samples | Raylib |

## Drawbacks

- Requiring every engine to have an input backend creates upfront work
  for each new engine integration.
- The strict header separation means developers must write both engine
  and Platform API calls — there is no unified "PlayOS Engine" wrapper.
- The overlay model (compositor captures engine frame) may not work
  with all rendering APIs (e.g., Vulkan direct-to-display without a
  compositable surface).

## Rationale and Alternatives

- **Unified "PlayOS Engine" API that wraps multiple engines:** Would
  simplify the developer experience but couples the platform to engine
  internals.  Rejected — violates engine-agnostic principle.
- **Platform API owns the window:** Would give the Platform API control
  over the swap chain but requires the Platform API to know about
  rendering APIs.  Rejected — the platform is not a rendering engine.
- **Callback-based input (engine pushes events to Platform API):**
  Simpler for engine authors but breaks the `Pressed`/`Down` semantics
  (the Platform API must poll, not receive events).  Rejected.
- **No integration kits; every developer writes their own bridging:**
  Increases fragmentation.  Integration kits provide a canonical,
  tested path for each major engine.

## Prior Art

- **SDL + platform APIs:** SDL provides its own input, windowing, and
  audio; applications call both SDL and platform-specific APIs (Win32,
  X11).  PlayOS follows this dual-API model.
- **Unity/Unreal platform abstraction:** Each engine has an internal
  platform abstraction layer.  PlayOS inverts this: the platform
  provides a single API that any engine can use.
- **Android NDK + game engines:** Android's `android_native_app_glue`
  provides the lifecycle loop; engines integrate with it.  PlayOS
  `Lifecycle::Update()` serves a similar role.
- **Steamworks API:** Valve's platform API is engine-agnostic; games
  call both their engine and Steamworks.  PlayOS follows this model
  but with a fully open specification.

## Unresolved Questions

- How does the overlay compositor capture the engine's frame for Vulkan
  applications that render directly to display?  A Vulkan layer or
  compositor extension may be needed.
- Should the input backend selection be runtime (capability-based) rather
  than compile-time (CMake)?  Compile-time is simpler but limits
  multi-backend binaries.
- How are multi-engine applications handled (e.g., game uses engine A
  but links a library that uses engine B)?  Out of scope for v1.0.

## Future Possibilities

- **Godot GDExtension:** A PlayOS Platform API module for Godot that
  exposes capabilities, input, storage, and cloud services as Godot
  nodes.
- **Engine-agnostic input layer:** A header-only C++ library that
  wraps the Platform API input in engine-friendly types (e.g.,
  `Vector2 GetLeftStick()` returning an engine-agnostic struct).
- **Pre-built integration kits:** Binary distributions of integration
  kits for popular engines, removing the need to build from source.
- **Vulkan layer for overlay capture:** A Vulkan layer that captures
  the final frame for compositor overlay without engine cooperation.
