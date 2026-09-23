# The Emulator Profile — design (Sprint 15, T7)

Status: implemented and verified 2026-09-22 (S15-T7). Related:
`sprints/Sprint-15.md`, `sdk-desktop-shim.md`, `dev-environment.md`.

## The problem

The `desktop` profile lets a developer iterate without hardware, but it runs a
*different build*: host `gcc`, upstream raylib, and `libplayos` with
`PLAYOS_BACKEND=stub`. The two things it cannot prove are exactly the things that
tend to break on the device:

- the **musl ABI** and the SDK cross toolchain — a glibc-linked binary silently
  does not run on the device;
- the **real platform** — raylib's `PLATFORM_PLAYOS` backend against the wlroots
  compositor, and `libplayos`' evdev backend against `playos-init`'s seat,
  lifecycle events and sandbox.

The alternative is flashing a USB stick and reading logs on the ROG Ally. The
`emulator` profile closes that gap: it boots the **device** build inside the
PlayOS QEMU image, so the real init, compositor, seat and sandbox are exercised
without hardware.

## What it is

Two pieces, one in each repository:

1. **`br2-external/configs/playos_emulator_defconfig`** (playos-refdistro) — a
   QEMU x86_64 boot image whose only job is to be a PlayOS session: init,
   compositor, shell, runtime, platform-api, raylib and overlay. It is slimmer
   than the QEMU dev defconfig — no BusyBox/Dropbear/installer/recovery/samples —
   because the game arrives on the data disk rather than inside the image.
2. **`scripts/build-emulator.sh`** (playos-tools, the SDK) — builds the game for
   the `device` profile, installs it onto a `playos-data` disk, boots the
   emulator image with it, and reports what it saw.

The image is built by `make emulator-build`; the QEMU side is
`scripts/emulator-run.sh` so the SDK script stays a thin front-end.

## Launching the game without a person: `playos.autostart`

Normally only the shell launches a game, and that needs somebody navigating the
library with a gamepad. Driving the shell's UI from outside is brittle (it
depends on screen layout and on the shell reading a synthetic evdev device), and
a hidden test that pokes at the UI is exactly the kind of thing that rots. So the
emulator uses an explicit boot hook, mirroring the existing `playos.install.auto`
token:

```
playos.autostart=<game-id>
```

When the token is present and the session is not in install or recovery mode,
init waits for `/proc/cmdline` to be readable, waits for the compositor, and then
performs the same sequence the shell's `LaunchGame` IPC request triggers:

1. generate a per-launch token and tell the compositor which game to expect
   (`SetExpectedGame`);
2. spawn the game under the Sprint 12 sandbox (`/data/games/<id>/manifest.json`,
   or the manifest path preserved by the caller);
3. emit `GameStarted` to the shell.

It is a **developer/testing hook, never a boot requirement**: if no token is
present nothing changes, and if the token names a game that is missing or
unrunnable init logs it and the session continues normally. Production images
never set it. It is not a console "auto-play" feature and does not alter the
shell's ownership of game launch.

## Disk assembly

`emulator-run.sh` builds a fresh ext4 image labelled `playos-data` (the label
init looks for), creates the game at `/data/games/<id>/` in the device layout
(executable + `manifest.json` + assets, `0755`), and boots QEMU with that image
as a virtio disk. Because `/data` already contains the game, init's first-boot
seeding leaves it alone, and the shell discovers it the same way it discovers a
seeded title.

## Verification

| Check | What it proves |
|---|---|
| serial log: `autostart requested - launching game <id>` | init honoured the token |
| serial log: `spawning game: <id> (/data/games/<id>/bin/game)` and no sandbox warnings | the musl device artifact passed manifest, exec and sandbox checks |
| compositor log `fps ... game=N` (N > 0), read out of the `playos-data` image with `debugfs` | the game rendered through `PLATFORM_PLAYOS` |
| `GameStarted` / lifecycle lines | the seat and trusted-IPC path work |

Input: the emulator kernel enables evdev and virtio-input, and the runner passes
a real host device through with `--gamepad /dev/input/eventN` (`-object
input-linux`). The automated check asserts only that the artifact launched and
that the compositor rendered it — controller fidelity through QEMU's synthetic
devices is a human check, because the device path expects a real gamepad rather
than a keyboard.

## Measured (2026-09-22)

A device-profile build of `playos-samples/bunnymark` (musl, `ld-musl-x86_64.so.1`)
through the SDK, booted with `make emulator-build`'s image:

- init: `autostart requested - launching game com.playos.sample-bunnymark`,
  then `spawning game: … (/data/games/…/bin/game)`; the S12 sandbox ran
  (Landlock is unsupported on the QEMU kernel, so it warned and continued, as
  designed).
- game: raylib `Platform backend: PLAYOS (Wayland + EGL/GLES2)`,
  `DISPLAY: Device initialized successfully`, softpipe GLES 3.1.
- compositor: `game surface added to scene (role 3)` and `fps shell=0 game=1`,
  with **zero** DRM atomic errors (`-vga none` + virtio-gpu; the default bochs
  card failed every atomic commit with "Out of memory").
- runner exit 0 with all three checks `[ok]`.

Two defects were found and fixed on the way: autostart used to fire before the
compositor control connection and the shell listener were registered (so
`SetExpectedGame` was dropped and the game rendered as an unclaimed surface), and
the guest's `playos-data` logs need their ext4 journal replayed (the runner now
does that on a private copy) because QEMU is killed rather than shut down.

## Decisions

- **Autostart lives in init, not the shell or the test script.** Init already
  owns game launch (`LaunchGame` is a thin parser over
  `playos_supervisor_spawn_game`), so the token is a few lines in the existing
  supervision loop and one shared launch helper. It also makes the hook reusable
  by any future headless harness.
- **A slim defconfig, not the dev image.** The emulator should be the smallest
  thing that can run a game: it reduces build time and makes the profile's
  dependencies explicit. The dev image keeps the debug tooling.
- **The game arrives on `/data`, not baked into the initramfs.** That keeps the
  emulator image game-agnostic and lets a developer swap artifacts without
  rebuilding the OS.

## Open questions

- **Gamepad fidelity under QEMU.** A synthetic keyboard proves the input path but
  not the controller database. Whether to wire QEMU gamepad passthrough into the
  default emulator command line is undecided.
- **GPU acceleration.** The default is Mesa softpipe (portable, slow). Whether to
  offer virtio-gpu/virgl for faster iteration depends on what developers run.
