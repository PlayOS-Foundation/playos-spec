# Security On-Device Self-Test Runbook

**Scope:** Sprint 12 on-device acceptance — verify the game sandbox behaves as designed on real hardware (ROG Ally), not just in host unit tests.

**Artifact:** `playos-init/tests/security_selftest.c` → `playos-security-selftest`, installed at `/usr/bin/playos-security-selftest`. It is a passive probe: it does not attempt to escape the sandbox, it reports what the current process is and is not allowed to do.

**Output:** one line per check to stderr, redirected by `playos-init` to `/data/log/game-security-selftest-stderr.log`, so results are on the persistent data partition (readable via USB or SSH).

---

## How to run

The self-test must run **as the game identity, inside the sandbox**. It is not useful run as root. There are two phases:

### Phase A — development image over SSH (fast iteration, full log capture)

1. Boot the **development** image (retains Dropbear SSH + a shell).
2. SSH in, then launch the self-test through the normal game spawn path:

   ```sh
   # The shell/CLI sends LaunchGame to playos-init; the sandbox is applied
   # before exec. The game id used here is arbitrary ("security-selftest").
   playos launch security-selftest
   # or, if a direct CLI isn't wired yet:
   printf '{"type":"LaunchGame","game_id":"security-selftest"}' \
     | socat - UNIX-CONNECT:/run/playos/control.sock
   ```

   For the LaunchGame path to find the binary, `/data/games/security-selftest/` needs a `manifest.json` whose `executable` points at `/usr/bin/playos-security-selftest` (a single-file "game" with no assets). If that's awkward on the dev image, a stopgap is to drop to a shell and simulate the identity manually — see "Manual identity simulation" below — but that is weaker evidence because it skips the actual spawn path.

3. Read the result:

   ```sh
   cat /data/log/game-security-selftest-stderr.log
   ```

### Phase B — production image manual smoke (no SSH, no shell)

The production image has no shell and no SSH, so the self-test can only run if a game manifest references it. The realistic Phase B checks are the manual smoke items in the next section; the self-test itself is primarily a Phase A / dev-image tool. Do not ship the self-test as a user-visible game.

### Manual identity simulation (dev-image stopgap only)

If the LaunchGame path isn't reachable, approximate the sandbox manually as root:

```sh
# NOT a substitute for the real spawn path — seccomp/Landlock ordering differs.
setpriv --reuid=1001 --regid=1001 --groups=108,29,102 \
  --no-new-privs /usr/bin/playos-security-selftest
```

Treat a PASS here as "identity probes work", never as proof of the full launch-time sandbox ordering.

---

## Probe interpretation

Each probe prints `[PASS]`, `[FAIL]`, or `[SKIP]`, plus an errno/value detail.

| Probe | Expected | What a FAIL means |
|---|---|---|
| `identity` | `uid=1001 gid=1001` | The game is still running as root or the wrong identity — privilege drop broken |
| `no_new_privs` | `no_new_privs=1` | `PR_SET_NO_NEW_PRIVS` was not applied before exec |
| `capabilities` | `eff=0x0 perm=0x0` | Capabilities remain — seccomp/capset ordering broken |
| `dri_card0_denied` | open fails (`EACCES`) | Game reached the primary DRM node → could modeset/take over display |
| `dri_render_allowed` | render node opens | EGL/GLES2 client rendering would fail |
| `input_allowed` | evdev node opens | Built-in controller backend would get no input |
| `audio_allowed` | ALSA PCM opens | Game audio would break |
| `control_sock_denied` | `connect` fails | Game reached the trusted control socket |
| `mount_denied` | `mount()` fails | seccomp deny-list not active |
| `config_denied` | `/data/config` open fails | Landlock allowlist too broad |
| `other_saves_denied` | another title's saves open fails | Landlock cross-game data leak |

`[SKIP]` (e.g. `renderD128 not present`, `no /dev/input/event* node`) means the node didn't exist on that boot — re-run on a fully-up ROG Ally where the GPU/input/audio devices are present.

---

## Known tensions to flag during the run

1. **`/dev/dri` Landlock rule — RESOLVED.** `landlock.c` now grants `/dev/dri` read-write (S12-T4), so `dri_render_allowed` PASSes on a fully-up Ally where `renderD128` exists. The primary node `/dev/dri/card0` stays denied by the `drm` group (Unix DAC), not by Landlock.

2. **`/dev/input` is granted by design — RESOLVED.** `input_allowed` PASSes because the game is in the `input` group and Landlock grants `/dev/input` read-only; the built-in controller works via raw evdev through `libplayos`. Reserved-button isolation is enforced by the `libplayos` snapshot mask + the compositor seat intercept, not by denying `/dev/input`. Sprint 12's spec has been reconciled to this.

3. **EC node separation — RESOLVED (Sprint 12).** A malicious game that bypasses `libplayos` and reads the raw evdev nodes directly could previously observe the reserved-button vendor/home nodes, because `/dev/input` was granted to the `input` group. `99-playos-input.rules` now moves the ASUS embedded-controller node (`Asus Keyboard`, `0b05:1abe`, event6/7/8 — Command Center `KEY_PROG1`/`KEY_PROG2`, M1/M2, brightness, volume, power/sleep) to `root:root 0600`, while the gamepad (`045e:028e`, event5) stays in `input`. HOME (`BTN_MODE`) lives on the gamepad node and is kept out of the game's snapshot by the `libplayos` mask + compositor seat intercept. On-device verification: after flashing, confirm event6/7/8 are `root:root 0600` and that a game still gets gamepad input but `EACCES` on the EC nodes.

4. **`mount_denied`, `config_denied`, `other_saves_denied`, `dri_card0_denied`, `control_sock_denied`, `identity`, `no_new_privs`, `capabilities` should all PASS** on the real spawn path. If any of these FAILs in Phase A, the sandbox ordering or allowlist is wrong.

---

## Manual smoke checks (production, no shell)

These need a human at the device; they complement the self-test:

- Boot the production image: it reaches the PlayOS shell, no shell prompt, no SSH.
- Launch a sample game: it renders, audio plays, controller works, and `B` returns to the shell cleanly.
- Press reserved buttons (`HOME`/command, volume up/down, M1/M2, power) while a game is focused: none of them appear as game input; they still drive shell/overlay/system behavior.
- Mount the USB afterward and inspect `/data/log/game-<id>-stderr.log` and `init.log` for sandbox warnings (e.g. "Landlock unsupported", "seccomp filter failed", "credential drop failed").
