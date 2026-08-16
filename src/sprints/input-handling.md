# ROG Ally Input Handling — Findings & Future Fixes

**Scope:** Findings from the Ally USB image retest (`playos-ally-usb.img`, built after the volume-node discovery + HOME/COMMAND latch changes). Logs were collected from the mounted `playos-data` partition (`/data/log/shell-stderr.log` and `/data/log/init.log`).

**Status:** Investigation only. No fixes applied in this pass.

---

## Discovered input topology (from shell logs)

- Gamepad: `Microsoft X-Box 360 pad` at `/dev/input/event5`.
- Vendor node: `Asus Keyboard` at `/dev/input/event8` (fd=9).
- Three `Asus Keyboard` nodes were skipped by the gamepad matcher as "missing stick axes": `event6`, `event7`, `event8`.
- The volume-node matcher reported success but selected **`event8` again** (fd=10) — the same physical node already open as the vendor node.

---

## Finding 1 — Volume node discovery found a duplicate, not the real volume node

Log lines:

```
found vendor node: 'Asus Keyboard' (/dev/input/event8) fd=9
scanning /dev/input/event* for volume node...
found volume node: 'Asus Keyboard' (/dev/input/event8) fd=10
```

The shell now opens `event8` twice and drains it twice. `event6` and `event7` (the two remaining `Asus Keyboard` nodes) are still never opened.

Because `event8` is drained by both the vendor and volume paths, every event on it is logged twice (`vendor raw ...` and `volume raw ...`). This double-drain is harmless functionally (both paths map to the same handling) but confirms the volume-node heuristic matched the wrong device.

---

## Finding 2 — Volume Up / Volume Down never reach the shell

Across the entire `shell-stderr.log` there are **zero** `KEY_VOLUMEUP` (`0x72`) or `KEY_VOLUMEDOWN` (`0x73`) EV_KEY events.

The only EV_KEY codes observed on `event8` are:

| Code | Linux key | PlayOS mapping |
|------|-----------|----------------|
| `0x94` | PROG1 | HOME / SYSTEM |
| `0xba` | F16 | COMMAND / QUICK_MENU |
| `0xbb` | F17 | (currently unmapped) |
| `0xbc` | F18 | (clears SYSTEM) |
| `0xb9` | F15 | (currently unmapped) |

Volume Up/Down are almost certainly emitted on `event6` and/or `event7`, which are never opened. The current `is_reserved_volume_device()` heuristic requires a volume capability bit plus an `Asus` name match, and it matches `event8` rather than the skipped nodes — so it is not proving that a distinct volume node exists, only that `event8` advertises (or is read as advertising) the capability.

---

## Finding 3 — HOME and COMMAND are momentary pulses, not held keys

Every `0x94` (HOME) and `0xba` (COMMAND) press observed is a pulse:

```
0x94 value=1
0x94 value=0   (~8ms later)
```

Press-to-release gap is consistently **7–9 ms** across all samples, with **no `value=2` autorepeat** events. The device itself releases almost immediately even if the button is physically held.

This matches the observed UI behavior ("flash for a split second" in Live Input Test). The 0.6s visual latch added for HOME/COMMAND is the right approach in spirit, but there is no held/level signal to extend naturally — these keys are momentary by hardware/firmware behavior.

The same pulse pattern applies to `0xbb` (F17) and `0xbc` (F18): `value=1` then `value=0` within ~8ms.

---

## Finding 4 — F15 behaves differently (level + autorepeat)

`0xb9` (F15) is the exception. One captured hold shows:

```
0xb9 value=1
0xb9 value=2   (~260ms later)
0xb9 value=2   (~40ms later)
0xb9 value=0
```

So F15 is a normal held key with autorepeat, unlike `0x94`/`0xba`/`0xbb`/`0xbc`. This is useful signal: not all vendor keys on `event8` are pulses.

---

## Finding 5 — Persistent logs do not enumerate input devices

`init.log` records lifecycle/IPC/audio events only. It does **not** dump `/proc/bus/input/devices` (or `evtest`/`libinput list-devices` output).

Consequence: from the persistent logs alone we cannot see the actual capabilities of `event6`/`event7` (names, EV_KEY bits, whether they advertise volume keys, and under which keycodes). A future diagnostic should capture `/proc/bus/input/devices` (and optionally per-device evbits) to the persistent log at startup.

---

## Finding 6 — shell-stderr.log contains two boot cycles

`shell-stderr.log` and `init.log` both contain two complete boot sequences (first boot `1.845s` → shutdown `165.5s`; second boot `1.862s` → shutdown `135.7s`). The logs are append-across-reboots, which is expected. When reading findings, the input init lines appear twice and the input test session starts around the second boot's `~41s` mark.

---

## Future TODO fixes

1. **Open all reserved `Asus Keyboard` nodes, not just the first volume-capability match.** Drain `event6`, `event7`, and `event8` separately (excluding the gamepad node). The current single-fd volume approach cannot cover split or capability-mismatched nodes.

2. **Add a startup input-device diagnostic to the persistent log.** Dump `/proc/bus/input/devices` (and ideally per-node EV_KEY evbits) at shell startup so `event6`/`event7` capabilities and volume keycodes are visible without hardware access.

3. **Determine where `KEY_VOLUMEUP`/`KEY_VOLUMEDOWN` actually land.** Once all `Asus Keyboard` nodes are drained, verify `0x72`/`0x73` (or alternate codes) appear and map them into the shell's volume handling. Handle the possibility that Up and Down are split across two nodes.

4. **Keep HOME/COMMAND as edge-triggered (pulse) inputs.** Do not rely on a held/level state for `0x94`/`0xba`/`0xbb`/`0xbc`. Use the visual latch (already added) and, for semantic actions, treat a press edge as the trigger. Only `0xb9` (F15) supports true hold/autorepeat semantics.

5. **Decide mapping for `0xbb` (F17) and `0xb9` (F15).** Both are currently unmapped; `0xbc` (F18) currently clears SYSTEM. Confirm the intended Ally button semantics (armoury/shortcut keys) and wire them into the shell/overlay key handling.

6. **Remove the double-drain once the real volume node is identified.** Avoid opening the same `event8` fd twice; the vendor path should remain the single drainer for vendor keycodes, and volume keys should come from their own node(s).

7. **Power button detection implemented (Sprint 9 follow-up).** `KEY_POWER` (`0x74`) and `KEY_SLEEP` (`0x8e`/142) on the ACPI Power/Sleep nodes are now decoded into `PLAYOS_BUTTON_POWER` (bit 16, reserved — games never see it). The shell discovers the power node (`is_reserved_power_device()`: `KEY_POWER`/`KEY_SLEEP`, no `BTN_SOUTH`/`BTN_MODE`), opens it without `EVIOCGRAB` so kernel ACPI power handling is unaffected, and exposes a `PWR` pill in the Live Input Test with the same 0.6s momentary-pulse latch as HOME/COMMAND. A new `shell_input_button_held()` level query was added and the Live Input Test now uses it for every pill. **Status:** implemented; compiles and builds, awaiting Ally hardware retest.

