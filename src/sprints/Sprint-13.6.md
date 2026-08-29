# Sprint 13.6 — Gamepad Identifier Database (SDL mappings)

**Goal:** Stop hand-coding per-controller quirks. Embed the community
SDL_GameControllerDB (Linux subset) into `playos-platform-api` and resolve every
connected gamepad through its identifier (bus/vendor/product/version) instead of
the fixed Xbox-standard evdev table, so all known controllers map to the correct
PlayOS logical buttons/axes out of the box.

**Primary Outcome:** A gamepad whose identifier exists in the database decodes
through its own mapping entry; unknown pads fall back to the Xbox-standard table;
the existing ROG Ally face-button quirk still wins over the database for the
Ally's internal controller.

**Prerequisites:** Sprint 13 landed (platform-api evdev backend + ROG Ally
face-button quirk); the raylib tree ships GLFW's embedded copy of the SDL2
database (`external/raylib/src/external/glfw/src/mappings.h`) to vendor from.

---

## Why This Sprint Exists

Sprint 13's Intel validation proved the *architecture* is portable, but the
evdev decoder still assumes every pad is an Xbox 360 pad. The ROG Ally's own
built-in controller is the proof: it reports a rotated X/Y and needed a
hand-written phys-based quirk. SDL's gamecontroller database already encodes
per-device bindings for thousands of pads; embedding it moves PlayOS from
"works with Xbox-layout pads" to "works with the pads the database knows".

## Decisions Locked

- **Database source:** the SDL2 `gamecontrollerdb.txt` lines shipped in GLFW's
  `mappings.h` (Linux entries only, i.e. entries carrying `platform:Linux`).
  Vendored into `playos-platform-api/src/backends/gamecontrollerdb.inc` as an
  auto-generated C string table. zlib-style license; attribution kept.
- **Identifier:** SDL GUID derived from `EVIOCGID` (bustype, vendor, product,
  version) using the SDL 2.0.5+ encoding; the device name is logged but not used
  for matching (matches SDL semantics).
- **Index semantics:** reproduce GLFW/SDL evdev enumeration exactly —
  buttons are indexed 0..N in increasing evdev code order from `BTN_MISC`;
  non-hat axes are indexed 0..N in increasing ABS code order; hats are indexed
  per pair (`ABS_HAT0X`, `ABS_HAT2X`, …). DB tokens are `b<idx>` (button),
  `a<idx>` (axis), `h<hat>.<bit>` (hat bit; 1=up, 2=right, 4=down, 8=left).
- **Precedence:** device-specific quirks (ROG Ally face-swap, reserved-button
  stripping) are applied to raw evdev codes **before** database resolution;
  database entry wins over the Xbox-standard fallback; no entry → fallback.
- **Scope:** both paths. The game path (`playos-platform-api`) and the trusted
  shell path (`playos-shell/src/input.c`) resolve the same database; the shell
  vendor-copies `gamepad_db.{c,h}` + `gamecontrollerdb.inc` so the trust
  boundary keeps the two decoders independent.
- **Unmapped SDL elements:** `misc1`, paddles and touchpad have no PlayOS
  logical equivalent yet — parsed and ignored (logged at debug level).

## Required Repository Changes

| Repo | Required work |
|---|---|
| `playos-platform-api` | Vendor `gamecontrollerdb.inc`; add `gamepad_db.{c,h}` (enumerate, GUID-match, parse, resolve); integrate into `backend_evdev.c`; host test parsing every embedded entry |
| `playos-spec` | This document + `SUMMARY.md`/`roadmap.md` links + `input-handling.md` DB layer note |

## Agent Task Breakdown

| Task ID | Task | Primary repo | Status |
|---|---|---|---|
| S13.6-T1 | Vendor Linux entries of the SDL2 DB as an embedded C table | `playos-platform-api` | done |
| S13.6-T2 | evdev enumeration + GUID matcher + mapping parser | `playos-platform-api` | done |
| S13.6-T3 | Resolve remap and use it in `backend_evdev.c` + shell `input.c` (fallback preserved) | `playos-platform-api` | done |
| S13.6-T4 | Host test: every embedded entry parses; Xbox fallback and Ally quirk precedence | `playos-platform-api` | done |
| S13.6-T5 | Spec docs + links | `playos-spec` | done |

## Acceptance Criteria

- [x] Every embedded DB entry parses (host test) and at least one non-Xbox entry resolves to a different binding than the fallback
- [x] Xbox 360 pad (045e:028e) still decodes A/B/X/Y correctly via its DB entry (host-tested SDL semantics)
- [ ] ROG Ally internal pad (045e:028e on phys `usb-0000:09:00.3-2`) decodes X/Y via its DB entry; face-swap quirk only patches the fallback path (awaiting on-device re-test)
- [x] Unknown pad (no DB entry) falls back to the Xbox-standard table (host-tested)
- [x] Host build + tests green; no public `playos_*.h` ABI change

## Exit Gate

Known controllers decode through their database entry; unknown controllers still
work via the Xbox-standard fallback; the ROG Ally quirk remains authoritative for
its internal pad.
