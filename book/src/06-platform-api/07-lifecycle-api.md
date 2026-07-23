# Lifecycle API

The `PlayOS::Lifecycle` module provides the minimum required to start,
advance, and stop the Platform API. It is gated by the required capability
`lifecycle.basic` — every conformant target provides it.

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
    if (!PlayOS::Lifecycle::Init()) {
        return 1; // initialization failed
    }

    while (running) {
        PlayOS::Lifecycle::Update();
        // read input, simulate, render
    }

    PlayOS::Lifecycle::Shutdown();
    return 0;
}
```

## Semantics

`Init()`
: Initializes the Platform API. Primes input polling so `Input::Pressed`
works on the first frame. Registers required capabilities with the
capability registry. Returns `false` if initialization fails (for example,
no input backend could be created on a platform that requires one).

`Update()`
: Advances the platform by one frame. Calls `Input::Update()` internally,
which polls the input backend and updates button edge-detection state.
Must complete in bounded time — no blocking I/O, no network calls.

`Shutdown()`
: Releases global resources allocated by `Init()`. Must be called before
the process exits. After `Shutdown()`, calling any Platform API function
is undefined.

## Capability

| Capability | Status | Required |
|---|---|---|
| `lifecycle.basic` | Required | Yes |

Since `lifecycle.basic` is required, applications may call `Init`,
`Update`, and `Shutdown` without a capability query.

## Relationship to input

`Lifecycle::Update` calls `Input::Update`. Applications using
`Lifecycle::Update` do not need to call `Input::Update` themselves.
The input state after `Update` reflects the current frame.

## Implementation note

The reference implementation (`playos/lifecycle.h`) keeps `Init` minimal —
input backend creation is deferred to the first `Input::Down` / `Pressed`
/ `GetAxis` call, triggered by `Update`.

## Related

- [Initialization and Shutdown](05-initialization-and-shutdown.md)
- [Event Loop and Update Model](06-event-loop-and-update-model.md)
- [Input API](09-input-api.md)
