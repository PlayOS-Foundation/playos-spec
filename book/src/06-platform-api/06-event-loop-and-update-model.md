# Event Loop and Update Model

The PlayOS Platform API uses a **polled** model: the application calls
`Lifecycle::Update()` once per frame, and the platform advances input,
capability state, and any other services that need per-frame work.

## The pattern

```cpp
Lifecycle::Init();
while (running) {
    Lifecycle::Update();     // polls input, advances platform state

    // Read input (these reflect the state after Update).
    if (Input::Pressed(Button::A)) { /* ... */ }
    if (Input::Pressed(Button::Home)) { /* ... */ }

    // ... render, simulate ...
}
Lifecycle::Shutdown();
```

## Why polled (not callbacks)

- The Platform API is designed to work inside any engine's loop — Raylib's
  `BeginDrawing`/`EndDrawing`, SDL's event loop, or a custom one.
- A polled model avoids threading or callback-registration complexity. It is
  simple, predictable, and the overhead of one poll per frame is negligible.
- `Pressed` (edge detection on button transitions) depends on comparing the
  current frame's state against the previous; a predictable per-frame update
  makes that comparison trivial and correct.

## Pressed vs Down

- `Input::Down(button)` — true **while the button is held**.
- `Input::Pressed(button)` — true only on the **frame** the button
transitions from up to down. This is the recommended way to trigger discrete
actions (launch, confirm, back).

`Lifecycle::Update` advances a two-frame buffer so that `Pressed` returns
true exactly once per physical press.

## Requirements

- `Update` MUST complete in bounded time (no blocking I/O, no network calls).
- `Update` MUST be safe to call at any framerate; high polling rates must not
  cause missed or duplicated button transitions.
- All input queries (`Down`, `Pressed`, `GetAxis`) MUST reflect the state
  after the most recent `Update` call.
