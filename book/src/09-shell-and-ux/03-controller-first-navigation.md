# Controller-First Navigation

The PlayOS Shell is **controller-first**. Every action must be reachable
with a gamepad. Keyboard, mouse, and touch are additive, never required.

## Logical buttons

Applications and the shell use logical PlayOS buttons, not device-specific
codes. Device profiles map physical inputs to these logical names.

```text
A, B, X, Y
DPadUp, DPadDown, DPadLeft, DPadRight
L1, R1, L2, R2
Start, Select
Home, QuickSettings
```

## Standard bindings

The reference shell uses console-conventional bindings:

| Input | Action |
|---|---|
| D-Pad / Left stick | Move focus |
| A | Confirm / select |
| B | Back / cancel |
| Home | Return to shell / open overlay |
| QuickSettings | Open quick settings |

`A = confirm` and `B = back` are fixed conventions in the reference shell.

## Navigation model

- Focus moves between discrete, visible elements; there is always exactly one
  focused element.
- Movement is directional (up/down/left/right), not pointer-based.
- The focused element is always clearly indicated.

## Requirements

- Every shell action MUST be reachable using only the gamepad.
- The shell MUST always show which element is focused.
- **Home** MUST return to the shell from anywhere, including a running game
  (enforced by the compositor — see
  [Input Routing](../08-runtime-architecture/10-input-routing.md)).
- Logical buttons MUST be used; the shell MUST NOT read device-specific
  input codes directly.

## Vertical slice note

Minimal implementation: D-Pad/stick moves a highlight across a short game
list, A launches the highlighted game, B cancels, and Home (Ally Armoury
button) returns from the game to the shell.

See: [Reference Shell](16-reference-shell.md),
[Input API](../06-platform-api/09-input-api.md),
[ROG Ally Reference](../12-device-model-and-porting/12-rog-ally-reference.md).
