# ADR-0011: Recovery Mode UI (Sprint 14 T6)

Status: Accepted (Sprint 14)

## Context

Fatal boot conditions (missing data partition, repeated compositor failure,
user request) previously halted with a console banner. Production readiness
requires a controller-friendly recovery UI reachable without a display server.

## Decision

- Recovery runs as a **shell mode** (`SCREEN_RECOVERY`) launched with
  `PLAYOS_RECOVERY=1`, reusing the existing compositor + shell stack.
- Entry points:
  1. Kernel cmdline `playos.recovery`
  2. **Volume Down held 5 seconds at boot** (evdev key-state scan, no boot
     delay when not held)
  3. Data partition missing / provisioning failure
  4. Compositor restart-limit exceeded
- Menu: Reboot, Shutdown, Factory Reset (trusted IPC), Rollback (toggle
  `boot.json` active slot), View Logs (`/data/log`).
- Normal fatal paths (e.g., shutdown IPC) retain the halt behavior.

## Consequences

- Recovery is usable with a gamepad on the same graphics stack as the shell.
- SimpleDRM/software rendering without AMDGPU is a follow-up for full
  graphics-independent recovery.
