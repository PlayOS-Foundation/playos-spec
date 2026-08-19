# PlayOS Testing Strategy

> **Cross-references:** [architecture.md](architecture.md) §19, [dev-environment.md](dev-environment.md), Sprint docs

---

## CI Layers

Testing is organized in seven layers, each building on the previous:

| Layer | Where | What |
|---|---|---|
| 1 | Host | Unit tests — logic, IPC serialization, manifest parsing |
| 2 | Host | Buildroot clean build — compilation and packaging |
| 3 | QEMU/OVMF | Boot test — UEFI boot to shell prompt |
| 4 | QEMU | Compositor + shell smoke test |
| 5 | QEMU | Game lifecycle integration test |
| 6 | ROG Ally | Physical device smoke test |
| 7 | ROG Ally | Long-running stability and update test |

**Layers 1–5 run on every PR and push to `main`.**  
**Layers 6–7 run on release candidates and scheduled nightly builds.**

---

## Layer 1 — Host Unit Tests

Each repository has a native host build with unit tests:

```bash
# Build and test on the host (no cross-compilation needed)
cmake -S playos-platform-api -B build-host -DPLAYOS_BACKEND=stub
cmake --build build-host
ctest --test-dir build-host --output-on-failure
```

**What to unit test:**

| Component | Tests |
|---|---|
| `playos-runtime` | IPC message serialization/deserialization; version mismatch handling; all message types round-trip |
| `playos-platform-api` | Input state bitmask helpers; manifest parser; storage path construction; lifecycle fd read |
| `playos-init` | Boot sequence logic; game launch validation; supervisor restart counter; shutdown sequence |
| `playos-compositor` | State machine transitions (all valid and invalid transitions); GPU selection logic |
| `playos-shell` | Manifest discovery and sorting; controller navigation logic; status bar data formatting |

**CI command:**
```yaml
- name: Unit tests
  run: |
    make setup
    make host-test  # runs ctest for all repos
```

---

## Layer 2 — Buildroot Clean Build

Verifies that the entire system compiles cleanly from source.

```yaml
- name: QEMU clean build
  run: |
    make qemu-config
    make qemu-build
```

Expected: no compiler errors or warnings (warnings-as-errors enabled in CI).

For PRs touching only a single component, the CI may optionally run a partial rebuild (`make -C buildroot O=build/qemu playos-compositor-rebuild`) for speed, but a full clean build runs on merge to `main`.

---

## Layer 3 — QEMU/OVMF Boot Test

Boots the built image in QEMU and verifies the system reaches a known-good state.

```bash
# scripts/ci-boot-test.sh
qemu-system-x86_64 \
  -machine q35 -cpu qemu64 -m 2G \
  -bios /usr/share/OVMF/OVMF_CODE.fd \
  -drive file=build/qemu/images/playos-esp.img,... \
  -drive file=build/qemu/images/data.img,... \
  -display none -serial file:boot-serial.log \
  -no-reboot &

# Wait for success marker in serial log (timeout: 120s)
timeout 120 grep -q "PLAYOS_BOOT_OK" boot-serial.log
```

`playos-init` writes `PLAYOS_BOOT_OK` to the serial console when:
- All virtual FSes are mounted
- Data partition is mounted
- Control IPC socket is ready
- Compositor readiness signal received

**Checks:**
- Boot completes within 30 seconds
- `playos-init` is PID 1 (verified via `/proc/1/comm` in serial log)
- No kernel panic or OOM killer invocation
- All mounts successful

---

## Layer 4 — Compositor + Shell Smoke Test

Runs the compositor in QEMU with a headless backend and verifies the Wayland session and shell start correctly.

```bash
# In the booted QEMU image (via serial or QEMU monitor):
# Verify Wayland socket exists
ls -la /run/playos/playos-0

# Verify compositor state via IPC
playos-ctl query-status
# Expected: compositor_state=SHELL_FOREGROUND

# Verify shell is alive
ps | grep playos-shell
```

**Checks:**
- `/run/playos/playos-0` socket exists
- `QueryStatus` returns `compositor_state=SHELL_FOREGROUND`
- `playos-shell` process is running
- No segfaults in compositor or shell logs

---

## Layer 5 — Game Lifecycle Integration Test

Runs the complete launch-background-resume-exit lifecycle with a stub game.

```bash
# Stub game binary: exits with code 0 after receiving TERMINATE lifecycle event
# Installed at /data/games/com.playos.ci-stub/

playos-ctl launch com.playos.ci-stub
sleep 2  # wait for GAME_FOREGROUND state

playos-ctl query-status
# Expected: compositor_state=GAME_FOREGROUND, game_id=com.playos.ci-stub

# Simulate System button (compositor-controlled in real hardware; inject via IPC in test)
playos-ctl simulate-system-button
sleep 0.5

playos-ctl query-status
# Expected: compositor_state=PLAYOS_UI_FOREGROUND_WITH_GAME_BACKGROUND

playos-ctl send-resume
sleep 0.5
playos-ctl query-status
# Expected: compositor_state=GAME_FOREGROUND

playos-ctl terminate-game com.playos.ci-stub
sleep 1
playos-ctl query-status
# Expected: compositor_state=SHELL_FOREGROUND, game_pid=null
```

**Crash recovery test:**
```bash
playos-ctl launch com.playos.ci-stub
sleep 2
kill -9 $(playos-ctl get-game-pid)
sleep 1
playos-ctl query-status
# Expected: compositor_state=SHELL_FOREGROUND (crash recovery in ≤500ms)
```

---

## Layer 6 — Physical ROG Ally Smoke Test

Run after every release candidate build. Document results in a test report.

### Boot tests
- [ ] Cold boot from power-off to shell: ≤ 5 seconds
- [ ] Repeated boot (×5): consistent boot time, no regressions
- [ ] Boot after unclean shutdown (simulated): filesystem check passes

### Display tests
- [ ] Shell renders at native resolution and refresh rate (1920×1080 @ 120Hz)
- [ ] No tearing or flickering visible during shell navigation
- [ ] Hardware-accelerated rendering confirmed (Mesa driver string in log)

### Controller input tests
- [ ] All D-pad directions navigate the shell grid
- [ ] A button selects; B button goes back
- [ ] Both analog sticks report correct axis values in `sample-input`
- [ ] Both triggers report correct values
- [ ] System button (ROG button / Armory Crate button) triggers overlay — not delivered to shell
- [ ] `evtest` on Ally controller shows expected event codes

### Game lifecycle tests
- [ ] `sample-triangle` launches and renders a colored triangle
- [ ] System button shows overlay above game; game is paused or background-throttled
- [ ] Resume returns to running game (no restart)
- [ ] Quit from overlay returns to shell
- [ ] `kill -9` on game PID: display returns to shell in ≤ 500ms
- [ ] Second launch attempt while game runs: rejected with error

### Audio tests
- [ ] `sample-audio` plays sine tone through speakers
- [ ] Plugging in headphones routes audio correctly
- [ ] Volume slider in overlay changes audible volume
- [ ] Game audio stops when overlay appears; resumes when dismissed

### Storage tests
- [ ] File written to save path survives reboot
- [ ] Game list updates when new game directory is added
- [ ] Cache clear via factory reset removes cache; system remains bootable

### Power tests
- [ ] Battery percentage shown and updates
- [ ] CPU and GPU temperatures shown in overlay
- [ ] Running CPU stress: thermal state changes to WARM; composite confirms in log
- [ ] Shutdown from overlay: clean poweroff

---

## Layer 7 — Long-Running Stability Test

Run on release candidates and nightly builds.

| Test | Duration | Pass criterion |
|---|---|---|
| Continuous shell idle | 4 hours | No crash, memory growth < 10 MB |
| Rapid launch/exit cycles | 100 iterations | All succeed; no compositor restart |
| System button cycles | 200 iterations | All transitions correct; no input loss |
| A/B update apply + rollback | Full cycle | New slot boots; rollback recovers correctly |
| Sustained GPU load (`sample-triangle`) | 30 minutes | No GPU hang, no thermal shutdown, stable FPS |

---

## Test Utilities

### `playos-ctl` — IPC test client

A development-only CLI tool (in `playos-tools`) that wraps the control IPC:

```
playos-ctl launch <game-id>
playos-ctl terminate <game-id> [--force]
playos-ctl query-status
playos-ctl get-game-pid
playos-ctl simulate-system-button    # inject system action via compositor debug socket
playos-ctl send-resume
playos-ctl shutdown
playos-ctl reboot
```

Not present in production builds.

### Useful third-party tools (development image)

| Tool | Purpose |
|---|---|
| `evtest` | Verify input device events |
| `modetest` | Verify DRM connectors and modes |
| `weston-info` | Inspect Wayland compositor |
| `aplay -l` | List ALSA devices |
| `speaker-test -t sine` | Test audio output |
| `stress-ng` | CPU/GPU stress for thermal testing |
| `perf stat` | CPU performance metrics |
| `apitrace` | Trace OpenGL calls (debugging) |
| `piglit` | OpenGL conformance tests |
