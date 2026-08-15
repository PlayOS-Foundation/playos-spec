# Sprint 15 — Game Developer SDK

**Goal:** Give third-party developers a self-contained SDK to build, run, and test a PlayOS game on a regular x86_64 Linux host — and iterate on Windows — without pulling the full Buildroot tree.

**Primary Outcome:** A downloadable `playos-sdk` (toolchain + `libplayos`/`libraylib` headers and libs) that turns standard `gcc`/`cmake` into a shippable `bin/game` + `manifest.json` + `assets/` artifact, with a working desktop testing loop.

**Prerequisites:** Sprint 14 complete — versioned, stable public `libplayos` C ABI (`PLAYOS_API_VERSION 1`) and stable Raylib backend ABI.

---

## Why This Sprint Exists

Today a game can only be built inside the Buildroot tree with the `x86_64-buildroot-linux-musl` toolchain, because the musl builds of `libplayos` and `libraylib` exist only there. The game ABI requires musl (not glibc), so a stock Ubuntu or glibc-linked binary won't run on device. This sprint packages that toolchain + libraries into a developer-facing SDK so third parties can build games independently, and provides a way to test them without ROG Ally hardware.

---

## Start Condition Checklist

- Sprint 14 complete: the public `libplayos` C ABI is frozen at `PLAYOS_API_VERSION 1` and the Raylib backend ABI is stable.
- `sdk-headers.tar.gz` exists from Sprint 14, but it is headers-only — it is not a full toolchain + library SDK.
- musl builds of `libplayos` and `libraylib` exist only inside the Buildroot output tree.
- `PLAYOS_BACKEND=stub` exists in `playos-platform-api` as the seed for the desktop host shim.
- The QEMU/container boot path exists from the Sprint 14 release pipeline's QEMU boot test suite.

---

## Decisions Locked for This Sprint

- **SDK artifact:** a self-contained `playos-sdk` download that does not require the full Buildroot tree.
- **Device ABI:** the shipped game binary must be musl-linked (`x86_64-buildroot-linux-musl`); a glibc-linked binary is not device-compatible.
- **Toolchain packaging:** a prebuilt `x86_64-buildroot-linux-musl` toolchain tarball, or an Alpine/musl base image.
- **Three build profiles:** `device`, `desktop`, and `emulator`.
- **Device backend:** `libraylib` is built with the `PLATFORM_PLAYOS` backend; `libplayos` uses the real evdev path.
- **Desktop shim:** the host `libplayos` shim is seeded from `PLAYOS_BACKEND=stub` and maps keyboard/gamepad to the controller ABI, no-op'ing lifecycle calls.
- **Emulator profile:** runs the `device` build inside the PlayOS QEMU/container image.
- **SDK home:** SDK packaging and tooling live in `playos-tools`.

---

## Scope

### In Scope

- Package the musl toolchain as a tarball and/or base image.
- Ship `libplayos` headers plus musl static/shared libraries at `PLAYOS_API_VERSION 1`.
- Ship musl `libraylib` headers and libraries built with `PLATFORM_PLAYOS`.
- Provide a CMake toolchain file and `pkg-config` files for the `device` profile.
- Build the `desktop` host shim seeded from `PLAYOS_BACKEND=stub`.
- Implement the `desktop` build profile (native `gcc` + raylib desktop backend + host shim).
- Implement the `emulator` build profile (device build running in QEMU/container).
- Build the reference sample entirely via the SDK and validate all three profiles.
- Document the SDK usage, profiles, and packaging layout.

### Explicitly Out of Scope

- Native Windows musl cross-toolchain — Windows iteration is via the `desktop` profile/shim.
- Store integration and SDK signing/distribution (post-MVP).
- Web, mobile, or other platform targets.
- Changes to the frozen `PLAYOS_API_VERSION 1` public ABI.

---

## Required Repository Changes

| Repo | Required work |
|---|---|
| `playos-tools` | Package the SDK: toolchain, headers/libs, CMake toolchain, `pkg-config`, profile scripts, SDK docs |
| `playos-platform-api` | Provide the `desktop` host shim seeded from `PLAYOS_BACKEND=stub`; ensure headers/libs are SDK-ready |
| `playos-shell` | Export the `PLATFORM_PLAYOS` Raylib backend build for SDK packaging |
| `playos-refdistro` | Extract musl `libplayos`/`libraylib` from Buildroot output; provide the QEMU/container `emulator` image |
| `playos-samples` | Build a reference sample entirely via the SDK and validate `device`, `desktop`, and `emulator` profiles |

---

## Expected Files and Directories

### `playos-tools`

```text
sdk/
    toolchain/                  # prebuilt x86_64-buildroot-linux-musl toolchain tarball or base image
    include/playos/             # libplayos public headers
    include/raylib.h            # libraylib headers
    lib/                        # musl libplayos.a/.so and libraylib.a/.so
    cmake/playos-toolchain.cmake
    pkgconfig/playos.pc
    pkgconfig/raylib-playos.pc
    scripts/build-device.sh
    scripts/build-desktop.sh
    scripts/build-emulator.sh
docs/sdk.md                     # SDK usage, profile matrix, artifact layout
```

### `playos-platform-api`

```text
src/desktop_shim.c              # host libplayos shim seeded from PLAYOS_BACKEND=stub
```

### `playos-shell`

```text
src/raylib/                     # PLATFORM_PLAYOS backend exported for SDK packaging
```

### `playos-refdistro`

```text
scripts/export-sdk.sh           # copies musl libplayos/libraylib from Buildroot output into the SDK tree
br2-external/configs/playos_emulator_defconfig   # QEMU/container boot image for the emulator profile
```

### `playos-samples`

```text
sdk-reference/                  # reference game built entirely via the SDK; exercises all three profiles
```

---

## Agent Task Breakdown

### Task Status Grid

| Task ID | Task | Primary repo | Status | Notes / evidence |
|---|---|---|---|---|
| S15-T1 | Package the musl toolchain tarball/base image | `playos-tools` | not started | |
| S15-T2 | Ship `libplayos` headers and musl static/shared libs | `playos-platform-api`, `playos-tools` | not started | |
| S15-T3 | Ship musl `libraylib` with the `PLATFORM_PLAYOS` backend | `playos-shell`, `playos-refdistro`, `playos-tools` | not started | |
| S15-T4 | Provide CMake toolchain and `pkg-config` for `device` | `playos-tools` | not started | |
| S15-T5 | Build the `desktop` host shim seeded from `PLAYOS_BACKEND=stub` | `playos-platform-api`, `playos-tools` | not started | |
| S15-T6 | Implement the `desktop` build profile | `playos-tools` | not started | |
| S15-T7 | Implement the `emulator` build profile | `playos-refdistro`, `playos-tools` | not started | |
| S15-T8 | Build the reference sample entirely via the SDK and validate all profiles | `playos-samples`, `playos-tools` | not started | |

Update the **Status** column as work progresses: `not started` → `in progress` → `blocked` or `done`.

### S15-T1 — Package the musl toolchain

Produce a self-contained `x86_64-buildroot-linux-musl` toolchain as a downloadable tarball, or provide an Alpine/musl base image that reproduces the same environment. The SDK must not require the full Buildroot tree to compile a game.

**Done when:** a fresh host with only the SDK toolchain installed can compile a minimal musl program that links and reports a musl ABI.

### S15-T2 — Ship `libplayos` headers and musl libraries

Package the `libplayos` public headers plus musl static and shared libraries into the SDK at `PLAYOS_API_VERSION 1`. The shared library keeps SONAME `libplayos.so.0`. Both the device (real evdev) and desktop (shim) variants must be selectable without changing the public headers.

**Done when:** `sdk/include/playos/` and `sdk/lib/libplayos.{a,so}` are present, and a game compiles against them with the SDK toolchain.

### S15-T3 — Ship musl `libraylib` with `PLATFORM_PLAYOS`

Build and package `libraylib` as a musl static/shared library with the `PLATFORM_PLAYOS` backend, and ship its headers in the SDK. Export the backend build from `playos-shell` and extract the artifacts from Buildroot output via `export-sdk.sh`.

**Done when:** `sdk/lib/libraylib.{a,so}` and `sdk/include/raylib.h` are present, and `sample-triangle`-equivalent code compiles and links against the packaged Raylib.

### S15-T4 — Provide CMake toolchain and `pkg-config`

Add `cmake/playos-toolchain.cmake` and `pkg-config` files (`playos.pc`, `raylib-playos.pc`) so a standard `cmake` or `gcc $(pkg-config --cflags --libs playos)` invocation produces a device-compatible musl binary.

**Done when:** `cmake` using the toolchain file and `pkg-config` using the `.pc` files both produce a valid musl device binary for the reference sample.

### S15-T5 — Build the `desktop` host shim

Implement the host `libplayos` shim in `playos-platform-api`, seeded from the existing `PLAYOS_BACKEND=stub`. The shim maps keyboard/gamepad input to the controller ABI and no-ops lifecycle calls, so a game runs unchanged in a normal desktop window. Package it as the `desktop`-profile `libplayos`.

**Done when:** a game linked against the shim runs on a Linux host, receives keyboard/gamepad input as controller state, and its lifecycle calls are safely no-op'ed.

### S15-T6 — Implement the `desktop` build profile

Add a `desktop` build profile that uses native `gcc`, Raylib's default desktop backend (X11/Wayland on Linux, Win32/GLFW on Windows), and the host shim from S15-T5. The same `game.c` used for `device` must build for `desktop` without source changes.

**Done when:** `scripts/build-desktop.sh` produces a native desktop binary from the same source as the `device` profile, and the game runs in a window with controller-equivalent input.

### S15-T7 — Implement the `emulator` build profile

Add an `emulator` profile that runs the `device` build inside the PlayOS QEMU/container image for high-fidelity testing without hardware. Wire the profile through `scripts/build-emulator.sh` and a QEMU/container image suitable for booting the device artifact.

**Done when:** `scripts/build-emulator.sh` boots the `device` artifact in QEMU/container and the game renders and accepts input in the emulated environment.

### S15-T8 — Validate the reference sample across all profiles

Build a reference sample in `playos-samples/sdk-reference/` entirely through the SDK, then build and run it against the `device`, `desktop`, and `emulator` profiles. Record the results for each profile.

**Done when:** the reference sample produces a valid musl `bin/game` for `device`, runs in a desktop window for `desktop`, boots and renders in QEMU for `emulator`, and the validation results are committed.

---

## Implementation Guidance

**The SDK is a packaging problem first.** The libraries already exist inside Buildroot; the work is extracting and organizing them so an external developer never sees the Buildroot tree.

**Keep the three profiles a single source, three link/build configurations.** Do not fork game code per profile; the whole point is that the same `game.c` builds everywhere.

**Seed the shim, don't write a new backend.** The desktop shim starts from `PLAYOS_BACKEND=stub`; it adds input mapping and lifecycle no-ops rather than inventing a parallel platform layer.

**Enforce musl for `device`.** A `device` build that silently falls back to glibc is a failure — verify the binary's interpreter/linkage as part of the profile scripts.

**Keep Windows out of the cross-toolchain scope.** Windows iteration goes through the `desktop` profile/shim; do not build a native Windows musl toolchain this sprint.

**Ship `pkg-config` and CMake parity.** Both paths must work, because different developers will prefer one; test both in the reference sample.

---

## Verification and Evidence

| Evidence | How it is produced |
|---|---|
| SDK is self-contained | Fresh-host build of a minimal game with only the SDK installed |
| Toolchain is musl | `file`/`readelf` on a produced binary shows the musl interpreter |
| `libplayos` packaged | `sdk/lib/libplayos.{a,so}` and headers present at `PLAYOS_API_VERSION 1` |
| `libraylib` packaged | `sdk/lib/libraylib.{a,so}` with `PLATFORM_PLAYOS` backend build |
| CMake path works | `cmake` build of the reference sample using `playos-toolchain.cmake` |
| `pkg-config` path works | `gcc $(pkg-config --cflags --libs playos)` produces a device binary |
| `desktop` shim works | Game runs in a window with keyboard/gamepad input on a Linux host |
| `desktop` profile works | Native desktop binary built from the same `game.c` |
| `emulator` profile works | Device artifact boots and renders in QEMU/container |
| Reference sample validated | Per-profile results committed in `playos-samples` |

---

## Acceptance Criteria

- [ ] A developer on a fresh x86_64 Ubuntu/Alpine host can produce a valid musl `bin/game` with only the SDK installed.
- [ ] The same `game.c` builds for `device`, `desktop`, and `emulator` profiles.
- [ ] The `desktop` profile runs the game in a windowed desktop environment on Linux (and, via a shim, Windows) with controller-equivalent input.
- [ ] The `emulator` profile boots the `device` artifact in QEMU and renders + accepts input.
- [ ] The SDK ships the musl toolchain, `libplayos` headers/libs at `PLAYOS_API_VERSION 1`, and musl `libraylib` with `PLATFORM_PLAYOS`.
- [ ] Both the CMake toolchain and `pkg-config` files produce a valid device binary.
- [ ] The reference sample is built entirely via the SDK and validated across all three profiles.
- [ ] The `device` binary is confirmed musl-linked, not glibc-linked.
- [ ] SDK usage, profile matrix, and artifact layout are documented in `playos-tools/docs/sdk.md`.

---

## Handoff to Sprint 16

Sprint 16 may assume:

- A downloadable `playos-sdk` ships the musl toolchain, `libplayos` (`PLAYOS_API_VERSION 1`), and musl `libraylib` with `PLATFORM_PLAYOS`.
- The `device`, `desktop`, and `emulator` build profiles all work from a single game source.
- The `desktop` host shim is seeded from `PLAYOS_BACKEND=stub` and maps keyboard/gamepad to the controller ABI.
- The `emulator` profile boots a `device` artifact in QEMU/container without hardware.
- The reference sample builds entirely via the SDK and validates all three profiles.
- The public `libplayos` ABI remains frozen at `PLAYOS_API_VERSION 1`.

---

## Exit Gate

A third-party developer can download `playos-sdk`, build a game for `device` with only the SDK installed, iterate on `desktop` without hardware, and validate on `emulator` in QEMU/container. The same source builds a shippable musl `bin/game` + `manifest.json` + `assets/` artifact across all three profiles.

*Previous: [Sprint 14](Sprint-14.md) | Next: [Sprint 16](Sprint-16.md)*
