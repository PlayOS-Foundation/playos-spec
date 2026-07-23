# SDL Integration

SDL (Simple DirectMedia Layer) is a widely-used cross-platform library
for windowing, input, and audio. Many game engines and frameworks are
built on SDL. Integrating SDL with the PlayOS Platform API follows the
same dual-include pattern as Raylib.

## Coexistence pattern

```cpp
#include <SDL.h>
#include "playos/playos.h"

int main() {
    SDL_Init(SDL_INIT_VIDEO | SDL_INIT_GAMEPAD);
    SDL_Window* window = SDL_CreateWindow("My Game", 1280, 720, 0);
    PlayOS::Lifecycle::Init();

    bool running = true;
    while (running) {
        PlayOS::Lifecycle::Update();

        // SDL events for window management, raw input.
        SDL_Event event;
        while (SDL_PollEvent(&event)) {
            if (event.type == SDL_EVENT_QUIT) running = false;
        }

        // Platform API for logical input and platform services.
        if (PlayOS::Input::Pressed(PlayOS::Button::Home)) { Pause(); }

        // SDL rendering...
    }

    PlayOS::Lifecycle::Shutdown();
    SDL_DestroyWindow(window);
    SDL_Quit();
}
```

## Input considerations

SDL has its own gamepad API (`SDL_GamepadButton`, `SDL_GetGamepadAxis`).
The Platform API also provides gamepad input via `Button` and `Axis`.
Developers may use either or both:

- **SDL gamepad API** — raw controller input, SDL's mapping database.
- **PlayOS Input API** — logical buttons and axes, device-profile mapping,
  `Pressed`/`Down` semantics.

Using the PlayOS Input API is recommended when the application needs
consistent `Home`/`QuickSettings` handling and the `Pressed` edge-detection
semantics. SDL input may be used alongside for low-level controller access.

## Input bridging

An SDL input bridge feeds SDL gamepad events into the Platform API input
system. This is a kit responsibility — see [Input Bridging](07-input-bridging.md).

## Build integration

```cmake
find_package(SDL3 REQUIRED)
target_link_libraries(my-game PRIVATE PlayOS::PlatformAPI SDL3::SDL3)
```

## Status

SDL integration is **planned**. A reference SDL integration kit and sample
are future work.

## Related

- [Engine-Agnostic Contract](02-engine-agnostic-contract.md)
- [Input Bridging](07-input-bridging.md)
- [Custom Engine Integration](05-custom-engine-integration.md)
