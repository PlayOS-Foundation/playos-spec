# The Desktop Shim — design (Sprint 15, T5/T6)

Status: design, agreed 2026-09-22. Implementation: S15-T5 (the shim), S15-T6 (the
profile). Related: `sprints/Sprint-15.md`, `playos-tools/docs/sdk.md`.

## The problem

A PlayOS game is a musl binary that talks to three things that only exist on the
device:

1. **`libplayos`** — the platform API (lifecycle, storage, power, system identity,
   audio, display policy, and a controller-state API).
2. **The PlayOS Raylib backend** — raylib built with `PLATFORM_PLAYOS`, a Wayland
   client that renders through the PlayOS compositor and gets its input from evdev.
3. **`playos-init`'s services** — the trusted control socket, `/data`, and the
   lifecycle events that tell a game when it is foreground, backgrounded or about to
   be suspended.

None of that exists on a developer's laptop. The goal of these tasks is that the
*same game source* builds and runs there, so a developer can iterate without
flashing hardware — and can see a window rather than reading logs.

## The two-ABI constraint

The device ABI is musl (`x86_64-buildroot-linux-musl`); a glibc binary cannot run on
the device, and the musl libraries cannot link into a glibc desktop process. So the
profiles are **separate builds of the same source**, not one binary run twice:

| Profile | Toolchain | Raylib | `libplayos` |
|---|---|---|---|
| `device` | musl (from the SDK) | `PLATFORM_PLAYOS` (from the SDK) | musl, `PLAYOS_BACKEND=evdev` |
| `desktop` | the host's `gcc` | **upstream raylib, default backend** | host build, `PLAYOS_BACKEND=stub` |
| `emulator` | musl (from the SDK) | `PLATFORM_PLAYOS` | musl, `PLAYOS_BACKEND=evdev`, run under QEMU |

The desktop profile therefore uses **raylib's own desktop backend** (X11/Wayland on
Linux, Win32/GLFW on Windows) rather than the PlayOS one. That is what makes a
window possible on a laptop, and it is why the shim does not have to pretend to be a
compositor.

## What the shim is

`libplayos` built natively with `PLAYOS_BACKEND=stub`: the same public headers, the
same API, but every PlayOS-only dependency replaced by a host-appropriate
implementation. It is *not* a new API and not a mocking framework — the game must
not need to know which profile it is in.

### Per-module contract

| Module | On the device | In the shim |
|---|---|---|
| `lifecycle` | talks to `playos-init`; foreground/background/suspend are real transitions | **Safe no-ops.** `playos_lifecycle_poll()` returns "unchanged", `wait*` returns immediately, and a game that never checks keeps running. |
| `input` | evdev, via `backend_evdev.c` + the gamepad database | **Best effort from the host's evdev** (see below); when no device is readable, it reports "no controller" and the game falls back to raylib input. |
| `storage` | `/data/...` on the device | `$XDG_DATA_HOME/playos` (or `~/.local/share/playos`), created on first use, so saves and settings persist across runs. |
| `power` | brightness/profile through `playos-init` | No-ops that report the last value they were given. |
| `system` | real device identity, firmware version | A stable synthetic identity so code that keys on it behaves consistently between runs. |
| `logging` | to the device's log files | stderr, honouring the same level filter. |
| `audio` | ALSA mixer control | No master-volume control (the host owns it); the calls succeed and report the current value. |
| `display` | the compositor owns display policy | Reports the host window's size/refresh; `set_vsync` is a no-op. |

### Input design (the interesting one)

On the desktop, **raylib is the primary input path** — it already reads gamepads and
keyboards through its own backend and the game is already written against it. The
shim's controller API exists for code that uses `libplayos` directly, so it is
best-effort:

- **Linux:** read the host's evdev devices. `backend_evdev.c` already does exactly
  this for the device, including the gamepad database, so the shim reuses it rather
  than reimplementing it. A developer in the `input` group gets their real gamepad.
- **Keyboard as a controller:** **WASD → left stick** (what games actually read),
  **arrows → d-pad** (menus), **Z/X/C/V → South/East/West/North** (A/B/X/Y),
  **Q/E → L1/R1**, **Enter/Backspace → Start/Select**. A development affordance,
  documented as such — not a device feature.
- **Not Linux (Windows via the emulator or a native build):** report "no
  controller". The game uses raylib input, which works everywhere. No fake input is
  synthesised — a shim that invents button presses is worse than one that reports
  nothing.

`backend_stub.c` today is a 20-line seed that reports "no controller" and returns
`-1`; T5 grows it into the above.

## What the shim deliberately does not do

- **No trusted IPC.** The control socket does not exist off-device; lifecycle calls
  become no-ops rather than failures, so games do not need `#ifdef`s.
- **No compositor, no overlay, no A/B slots, no real `boot.json`.** Anything that
  depends on those is device-only behaviour.
- **No device identity spoofing beyond a stable synthetic value.** Code that gates
  on "am I on a PlayOS device" should see something consistent, not a lie that
  changes between runs.
- **No performance or thermal fidelity.** A laptop does not have the Ally's
  power/thermal profile, and pretending otherwise would mislead.

## Verification plan

What each check can actually prove, and where:

| Check | Needs a display? | What it proves |
|---|---|---|
| `cmake -DPLAYOS_BACKEND=stub` builds natively | no | the shim compiles with the host toolchain (done: builds on the workstation) |
| a sample links against the shim and starts | no | the API is complete enough to link and reach `main` |
| lifecycle calls return promptly and do not block | no | the no-op contract — this is what would otherwise hang a desktop game |
| a window opens, keyboard and gamepad control the game | **yes** | the desktop profile end to end (T6) |

A headless CI host can therefore prove the first three; the window check is the one
that needs a machine with a display — and it is the check a developer performs on
their own laptop anyway.

## Open questions

- **raylib for the desktop profile:** require a system-installed raylib, or have the
  SDK fetch/build upstream raylib once? (T6 has to answer this; a vendored build is
  more reproducible, a system package is faster to start.)
- **Where the shim's storage root lives on Windows** when the emulator profile is
  used from WSL — inside WSL, or on the Windows side so saves move between the two?
- **Audio:** do we drive the host's mixer, or leave it entirely to the host? The
  current design leaves it to the host.
