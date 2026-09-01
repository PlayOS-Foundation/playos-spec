# Building Your First PlayOS Game

A PlayOS game is a regular C (or C++) binary that targets the frozen public
`libplayos` API (`PLAYOS_API_VERSION 1`) and renders through Raylib's
`PLATFORM_PLAYOS` backend.

The fastest path is the getting-started guide shipped with the SDK:

- [`playos-platform-api/docs/getting-started.md`](https://github.com/PlayOS-Foundation/playos-platform-api/blob/main/docs/getting-started.md)
- [`examples/minimal_game.c`](https://github.com/PlayOS-Foundation/playos-platform-api/blob/main/examples/minimal_game.c)

## Game package layout

A shippable game is a directory on the data partition:

```text
bin/game            # musl-linked ELF (device ABI)
manifest.json       # game id, entry, permissions
assets/             # art, audio, data
```

`manifest.json` is signed (development key in the MVP) and verified by
`playos-init` before launch.

## Build profiles

- **device** — musl toolchain + `libplayos` + Raylib `PLATFORM_PLAYOS` (Sprint 15 SDK)
- **desktop** — native gcc + Raylib desktop backend + `libplayos` host shim (Sprint 15)
- **emulator** — the device build inside PlayOS QEMU (Sprint 15)

## The console contract

- Handle lifecycle events every frame (foreground/background/suspend/terminate).
- Poll input from `libplayos`; never read evdev directly.
- Save through `libplayos` storage APIs (crash-safe, per-game isolation).
- Log through `libplayos` logging (persisted per session).

See the following guides for each contract area.
