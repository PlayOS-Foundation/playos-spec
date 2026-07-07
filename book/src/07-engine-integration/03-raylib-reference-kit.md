# Raylib Reference Kit

Raylib is the **reference engine** for PlayOS. The shell, the samples, the
tutorials, and the starter templates all use it — but the Platform API itself
does not depend on it.

## Why Raylib

Raylib was chosen as the reference because it shares the same philosophy as
PlayOS: small, fast, easy, cross-platform, written in C (usable from C++),
and open source. It is simple enough that a developer can understand the
entire engine, and approachable enough that tutorials stay short.

## How they coexist

The [`hello-playos`](https://github.com/PlayOS-Foundation/playos-samples/blob/main/hello-playos/src/main.cpp)
sample and the
[PlayOS Shell](https://github.com/PlayOS-Foundation/playos-shell/blob/main/src/main.cpp)
show the correct pattern:

```cpp
#include "raylib.h"           // rendering
#include "playos/playos.h"    // platform services

int main() {
    InitWindow(1280, 720, "My Game");
    SetTargetFPS(60);
    PlayOS::Lifecycle::Init();

    while (!WindowShouldClose()) {
        PlayOS::Lifecycle::Update();

        // Platform services via the Platform API.
        if (PlayOS::Input::Pressed(PlayOS::Button::A)) { Confirm(); }

        // Rendering via Raylib.
        BeginDrawing();
        ClearBackground(BLACK);
        DrawText("Hello", 10, 10, 20, WHITE);
        EndDrawing();
    }

    PlayOS::Lifecycle::Shutdown();
    CloseWindow();
}
```

**Raylib does not provide Platform API services; the Platform API does not
provide rendering.** Each stays in its lane.

## Build integration

The reference CMake pattern fetches Raylib via `FetchContent` and links the
Platform API as a sibling dependency or GitHub fetch:

```cmake
FetchContent_Declare(raylib GIT_REPOSITORY https://github.com/raysan5/raylib.git GIT_TAG 5.5)
FetchContent_MakeAvailable(raylib)

# Platform API resolved from sibling checkout or GitHub
target_link_libraries(my-game PRIVATE PlayOS::PlatformAPI raylib)
```

## Where Raylib is used in PlayOS

| Component | Repository | How Raylib is used |
|---|---|---|
| Reference Shell | `playos-shell` | Full reference console UI. As a Wayland client on runtime devices. FetchContent build. |
| `hello-playos` sample | `playos-samples` | Minimal Raylib + Platform API game. |
| Tutorials and starter templates | `playos-samples` (future) | Teaching materials. |

## Requirements

- The Platform API MUST NOT require Raylib to build or function.
- Samples and the reference shell MAY hard-code Raylib (they are Raylib-first
  by design).
- The kit MUST document the CMake `FetchContent` pattern so any developer can
  adopt it.
- Alternative engine kits (SDL, Godot, custom) MUST demonstrate the same
  dual-include pattern (`engine.h` + `playos/playos.h`).
