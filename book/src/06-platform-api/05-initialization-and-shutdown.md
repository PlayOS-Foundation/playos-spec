# Initialization and Shutdown

The `PlayOS::Lifecycle` module provides the minimum required to start and
stop the Platform API.

## API

```cpp
namespace PlayOS {
namespace Lifecycle {

bool Init();      // Initialize the platform (input backend, capabilities).
void Update();    // Advance one frame, poll input.
void Shutdown();  // Release platform resources.

} // namespace Lifecycle
} // namespace PlayOS
```

## Usage

```cpp
#include "playos/playos.h"

int main() {
    PlayOS::Lifecycle::Init();

    while (!WindowShouldClose()) {       // Raylib or engine loop
        PlayOS::Lifecycle::Update();     // advances input, capabilities
        // ... game logic ...
    }

    PlayOS::Lifecycle::Shutdown();
    return 0;
}
```

## Requirements

- `Init` MUST prime input polling so `Input::Pressed` works on the first
  frame.
- `Update` MUST call the underlying input backend's poll, then update the
  button edge-detection state (`Pressed` vs `Down`).
- `Init` MUST return `false` if initialization fails (for example, no input
  backend could be created).
- `Shutdown` MUST release any global resources allocated by `Init`.
- Callers MUST call `Shutdown` before the process exits.

## Relationship to input

`Lifecycle::Update` calls `Input::Update`, which delegates to the
platform-selected backend. Applications do not need to call
`Input::Update` themselves — `Lifecycle::Update` suffices. See
[Event Loop and Update Model](06-event-loop-and-update-model.md).

## Backend initialization

The first call to `Input::Down` / `Pressed` / `GetAxis` after `Init` creates
the platform-specific input backend (XInput on Windows, evdev on Linux, a
null backend elsewhere). This happens lazily so `Init` itself has minimal
work.
