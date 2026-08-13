# Sprint 15 — Game Developer SDK

**Goal:** Give third-party developers a self-contained SDK to build, run, and test a PlayOS game on a regular x86_64 Linux host — and iterate on Windows — without pulling the full Buildroot tree.

**Primary Outcome:** A downloadable `playos-sdk` (toolchain + `libplayos`/`libraylib` headers and libs) that turns standard `gcc`/`cmake` into a shippable `bin/game` + `manifest.json` + `assets/` artifact, with a working desktop testing loop.

**Status:** 🟡 Placeholder — not started. Scope captured for planning; see [`post-mvp.md`](../post-mvp.md) §Game Developer SDK (`playos-sdk`) for the full motivation and testing story.

**Prerequisites:** Sprint 14 complete — versioned, stable public `libplayos` C ABI (`PLAYOS_API_VERSION 1`) and stable Raylib backend ABI.

---

## Why This Sprint Exists

Today a game can only be built inside the Buildroot tree with the `x86_64-buildroot-linux-musl` toolchain, because the musl builds of `libplayos` and `libraylib` exist only there. The game ABI requires musl (not glibc), so a stock Ubuntu or glibc-linked binary won't run on device. This sprint packages that toolchain + libraries into a developer-facing SDK so third parties can build games independently, and provides a way to test them without ROG Ally hardware.

---

## Key Deliverables (placeholder — to be decomposed)

### SDK artifact

- Prebuilt `x86_64-buildroot-linux-musl` toolchain (or an Alpine/musl base image).
- `libplayos` headers + static/shared musl libs.
- `libraylib` headers + libs (musl) with the `PLATFORM_PLAYOS` backend.
- CMake toolchain file + `pkg-config` files.

### Three build profiles

1. **`device`** — musl + `PLATFORM_PLAYOS` raylib backend + real `libplayos` (evdev). The shipped artifact.
2. **`desktop`** — native `gcc` + raylib's default desktop backend (X11/Wayland on Linux, Win32/GLFW on Windows) + a host `libplayos` shim (maps keyboard/gamepad to the controller ABI, no-ops lifecycle). Runs the game in a normal desktop window. Seed: existing `PLAYOS_BACKEND=stub`.
3. **`emulator`** — run the `device` build inside the PlayOS QEMU/container image for high-fidelity testing without hardware.

### Acceptance criteria (draft)

- [ ] A developer on a fresh x86_64 Ubuntu/Alpine host can produce a valid musl `bin/game` with only the SDK installed.
- [ ] The same `game.c` builds for `device`, `desktop`, and `emulator` profiles.
- [ ] The `desktop` profile runs the game in a windowed desktop environment on Linux (and, via a shim, Windows) with controller-equivalent input.
- [ ] The `emulator` profile boots the `device` artifact in QEMU and renders + accepts input.

---

*Decompose into tasks (S15-T1…) when this sprint is scheduled.*
