# Game Input

Games receive hardware-agnostic logical controller state through `libplayos`.

## API

```c
if (playos_input_controller_connected()) {
    PlayOSControllerState state;
    if (playos_input_get_controller_state(&state) == 0) {
        if (playos_input_button_down(&state, PLAYOS_BUTTON_SOUTH)) { /* A */ }
        float lx = state.axes[PLAYOS_AXIS_LEFT_X];
    }
}
```

## Logical layout

| PlayOS | Xbox | PlayStation |
|---|---|---|
| `PLAYOS_BUTTON_SOUTH` | A | Cross |
| `PLAYOS_BUTTON_EAST` | B | Circle |
| `PLAYOS_BUTTON_WEST` | X | Square |
| `PLAYOS_BUTTON_NORTH` | Y | Triangle |

Axes: `PLAYOS_AXIS_LEFT_X/Y`, `PLAYOS_AXIS_RIGHT_X/Y` (`[-1.0, 1.0]`),
`PLAYOS_AXIS_LEFT_TRIGGER/RIGHT_TRIGGER` (`[0.0, 1.0]`).

## Reserved buttons

`PLAYOS_BUTTON_SYSTEM`, `PLAYOS_BUTTON_QUICK_MENU`, and `PLAYOS_BUTTON_POWER`
are reserved for the shell and are **never delivered to game processes**.

## Controller database

PlayOS embeds the SDL GameControllerDB mappings so physical controllers map to
the logical layout above. The ROG Ally internal controller is handled
correctly without special-casing in the game.
