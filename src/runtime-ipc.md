# PlayOS Runtime IPC Specification

> **Authoritative repository:** `playos-runtime`  
> **Protocol version:** 1  
> **Cross-references:** [architecture.md](architecture.md) §7.7, [security-model.md](security-model.md)

This document specifies the **internal** PlayOS IPC protocol. It is **not** a public application interface. Games and non-trusted clients must never connect to these endpoints.

---

## Table of Contents

1. [Overview](#1-overview)
2. [Transport](#2-transport)
3. [Access Control](#3-access-control)
4. [Message Format](#4-message-format)
5. [Control IPC — `control.sock`](#5-control-ipc--controlsock)
6. [Lifecycle Transport — per-game fd](#6-lifecycle-transport--per-game-fd)
7. [Compositor Control Channel](#7-compositor-control-channel)
8. [Protocol Versioning](#8-protocol-versioning)
9. [Error Handling](#9-error-handling)

---

## 1. Overview

`playos-runtime` owns the private integration layer between trusted PlayOS system components. It defines three distinct communication channels:

| Channel | Transport | Direction | Purpose |
|---|---|---|---|
| Control IPC | Unix socket | Shell/overlay → `playos-init` | Game launch, shutdown, factory reset, system commands |
| Lifecycle transport | Pipe fd per game | `playos-init` → game | Lifecycle events (foreground, background, terminate) |
| Compositor control | Unix socket | `playos-init`/runtime → compositor | Set expected game, show/hide overlay, force game exit |

Normal games never use these channels directly. They receive lifecycle events through `playos_lifecycle_poll()` from `playos-platform-api`, which reads from the lifecycle fd.

---

## 2. Transport

### Control IPC socket
```
/run/playos/control.sock
Type: SOCK_SEQPACKET (message-boundaries preserved, reliable, ordered)
Owner: root:playos-trusted
Mode: 0660
```

### Compositor control socket
```
/run/playos/compositor.sock
Type: SOCK_SEQPACKET
Owner: root:playos-trusted
Mode: 0660
```

### Lifecycle fd
A write end of a `pipe(2)` passed to the game process as `PLAYOS_LIFECYCLE_FD` in its environment. The game process owns the read end; `playos-init` writes events to the write end.

---

## 3. Access Control

Only processes in the `playos-trusted` UNIX group may connect to control sockets.

| Component | Group membership |
|---|---|
| `playos-init` | root (owns sockets) |
| `playos-compositor` | `playos-trusted` |
| `playos-shell` | `playos-trusted` |
| `playos-overlay` | `playos-trusted` |
| Active game | **Not in `playos-trusted`** — no socket access |

The lifecycle fd is a one-directional pipe: the game can only read from it, never write to it or connect to any IPC socket.

---

## 4. Message Format

All messages use a simple length-prefixed binary frame:

```
+--------+--------+---...---+
| magic  | length |  body   |
| 4 bytes| 4 bytes| N bytes |
+--------+--------+---...---+
```

- **magic**: `0x504C4F53` (`PLOS` in ASCII) — validates frame start
- **length**: little-endian uint32, byte count of body only
- **body**: JSON-encoded message (UTF-8, no trailing null)

Using JSON for the body makes messages human-readable for debugging while keeping the framing simple and versioned.

**Maximum message size:** 65536 bytes (64 KB)

**Example message:**
```json
{
  "v": 1,
  "type": "LaunchGame",
  "game_id": "com.example.game",
  "manifest_path": "/data/games/com.example.game/manifest.json"
}
```

All messages include `"v"` (protocol version) and `"type"` fields.

---

## 5. Control IPC — `control.sock`

Trusted clients (shell, overlay) send requests; `playos-init` sends responses and async events.

### Request → Response messages

#### `LaunchGame`
```json
{
  "v": 1,
  "type": "LaunchGame",
  "game_id": "com.example.game",
  "manifest_path": "/data/games/com.example.game/manifest.json"
}
```
**Response:**
```json
{ "v": 1, "type": "LaunchGameAck", "game_id": "com.example.game", "launch_token": "<uuid>" }
{ "v": 1, "type": "LaunchGameError", "game_id": "com.example.game", "reason": "already_running" }
```
**Error reasons:** `already_running`, `invalid_manifest`, `executable_not_found`, `unsupported_api_version`, `permission_denied`

---

#### `TerminateGame`
```json
{ "v": 1, "type": "TerminateGame", "game_id": "com.example.game", "force": false }
```
`force: true` — skip cooperative SIGTERM and go straight to SIGKILL after 500ms.

**Response:**
```json
{ "v": 1, "type": "TerminateGameAck", "game_id": "com.example.game" }
```

---

#### `QueryStatus`
```json
{ "v": 1, "type": "QueryStatus" }
```
**Response:**
```json
{
  "v": 1,
  "type": "StatusReport",
  "compositor_pid": 42,
  "compositor_state": "GAME_FOREGROUND",
  "game_pid": 123,
  "game_id": "com.example.game",
  "uptime_s": 3600
}
```
`game_pid` and `game_id` are `null` when no game is running.

---

#### `Shutdown`
```json
{ "v": 1, "type": "Shutdown" }
```
`playos-init` delivers `PLAYOS_LIFECYCLE_TERMINATE` to the game, waits up to 2 seconds, then kills all processes, syncs filesystems, and calls `reboot(RB_POWER_OFF)`.

---

#### `Reboot`
```json
{ "v": 1, "type": "Reboot" }
```
Same as `Shutdown` but calls `reboot(RB_AUTOBOOT)`.

---

#### `FactoryReset`
```json
{
  "v": 1,
  "type": "FactoryReset",
  "erase_games": false,
  "erase_saves": false,
  "erase_cache": true,
  "erase_config": true,
  "erase_logs": false
}
```
Requires no active game. Erases selected `/data/` subdirectories and recreates them.

**Response:**
```json
{ "v": 1, "type": "FactoryResetComplete" }
{ "v": 1, "type": "FactoryResetError", "reason": "game_running" }
```

---

#### `RollbackSlot`
```json
{ "v": 1, "type": "RollbackSlot" }
```
Recovery-menu action (S14). `playos-init` owns the A/B boot metadata and applies the full rollback to `/EFI/playos/boot.json` — the current slot is marked `bad`, the target slot becomes `pending`, and the target slot's boot count is reset — then reboots. No response is sent: on error `playos-init` logs the reason and leaves `boot.json` untouched so the running session can report the failure. Clients (the shell recovery menu) must not rewrite `boot.json` directly.

---

#### `StartInstaller`
```json
{ "v": 1, "type": "StartInstaller", "target_disk": "/dev/nvme0n1" }
{ "v": 1, "type": "StartInstallerAck" }
{ "v": 1, "type": "StartInstallerError", "reason": "umount_failed" }
```
Hand the session over to the runtime installer (S13.7; extended by S14-T10 and the S14 installer-UX pass). The shell's installer front-end picks the target disk and sends it here; `playos-init` then performs a **seamless handoff**: the kernel console is suppressed and the VT blanked, the shell and overlay are stopped (the installer becomes the foreground client) and waited for, and the dev SSH key is copied to `/tmp`. The **compositor keeps running** and **`/data` stays mounted** — stopping the compositor only served to unmount `/data`, which cost a DRM modeset blink and left the installer with no log; `/data` holds the boot medium, never the target, because the picker refuses the disk the system booted from. Only the target is released: `/EFI` is unmounted **only when it sits on the target disk** (`playos_mount_is_on_target()`), so nothing has to be restored when installing to a different disk. `playos-installer` is spawned with `PLAYOS_INSTALL_TARGET=<target_disk>`, which makes it skip its own picker and confirmation and start the destructive phase directly. `target_disk` is optional — when omitted, or when it matches no enumerated disk, the installer falls back to its interactive picker. On success the installer reboots; on failure init respawns shell/overlay against the still-running compositor, restores `/EFI`, and the installer's log survives in `/data/log/installer.log`. `StartInstallerAck`/`StartInstallerError` report only whether the handoff itself could be set up.

---

#### `PrepareInstall`
```json
{ "v": 1, "type": "PrepareInstall", "target_disk": "/dev/nvme0n1" }
{ "v": 1, "type": "PrepareInstallAck" }
{ "v": 1, "type": "PrepareInstallError" }
```
Added by Sprint 14.5-T2. Validate an install target and release its mounts **before** the shell commits to a progress screen, so a target that cannot be installed fails while the user is still on the picker. `playos-init` rejects the request when the target is not present, or when it holds the running root (`/`) or `/data` — installing then would erase the system in use — and otherwise unmounts the target's own `/EFI` when the ESP sits on it (`playos_mount_is_on_target()`), because a mounted ESP makes `mkfs` refuse the target. The install engine repeats the release in step 0: this is the early answer, not a replacement. `PrepareInstallAck`/`PrepareInstallError` report only the check.

---

#### `InstallProgress`, `InstallComplete`, `InstallError`
```json
{ "v": 1, "type": "InstallProgress", "step": 3, "step_name": "Reserve system B", "percent": 37 }
{ "v": 1, "type": "InstallComplete" }
{ "v": 1, "type": "InstallError", "step": 3, "reason": "..." }
```
Added by Sprint 14.5-T2. These are **events, not requests**: the screen-less `playos-install-worker` (S14.5-T3) reports them like any other client and `playos-init` relays them to the registered shell listener — the same path `UpdateProgress` takes, and the only available one, because the shell's listener is a connection *to* init rather than a server the worker could dial. The shell therefore never talks to the worker. Init forwards only the fields that follow the message's `type` and re-adds `v`/`type` itself, so a worker's payload goes after the type.

---

#### `SetPerfProfile`
```json
{ "v": 1, "type": "SetPerfProfile", "profile": "balanced" }
```
`profile` values: `"balanced"`, `"power_save"`, `"performance"`

**Response:**
```json
{ "v": 1, "type": "SetPerfProfile", "accepted": true }
{ "v": 1, "type": "SetPerfProfile", "accepted": false, "reason": "thermal_denied" }
```
`reason` values: `thermal_denied`, `invalid_profile`, `epp_write_failed`

---

#### `Suspend`
```json
{ "v": 1, "type": "Suspend" }
```
Fire-and-forget. `playos-init` delivers `PLAYOS_LIFECYCLE_SUSPEND` to the active game, attempts S3 suspend (`mem` to `/sys/power/state`), then delivers `PLAYOS_LIFECYCLE_RESUME` after resume (or immediately on failure). No response is sent.

---

#### `ApplyUpdate`
```json
{ "v": 1, "type": "ApplyUpdate", "path": "/data/updates/0.2.0.playosb" }
```
Requests that `playos-init` apply a system update bundle at `path` to the inactive slot. `path` must reside under `/data/updates/` and carry the `.playosb` suffix. Exactly one update may be in flight at a time.

**Response:**
```json
{ "v": 1, "type": "ApplyUpdateAck", "accepted": true }
{ "v": 1, "type": "ApplyUpdateError", "reason": "..." }
```
**Error reasons:** `not_found`, `invalid_bundle`, `signature_invalid`, `update_in_progress`, `game_running`, `internal_error`

Progress is reported via the async `UpdateProgress` / `UpdateComplete` / `UpdateError` events below.

---

### Async events (init → client, unsolicited)

#### `GameStarted`
```json
{
  "v": 1,
  "type": "GameStarted",
  "game_id": "com.example.game",
  "pid": 456,
  "launch_token": "<uuid>"
}
```

#### `GameExited`
```json
{
  "v": 1,
  "type": "GameExited",
  "game_id": "com.example.game",
  "exit_code": 0
}
```

#### `GameCrashed`
```json
{
  "v": 1,
  "type": "GameCrashed",
  "game_id": "com.example.game",
  "exit_code": 134,
  "signal": 6
}
```

#### `ThermalStateChanged`
```json
{ "v": 1, "type": "ThermalStateChanged", "state": 2 }
```
`state` values (integer): `0` normal, `1` warm, `2` hot, `3` critical

#### `PerfProfileChanged`
```json
{ "v": 1, "type": "PerfProfileChanged", "profile": 1 }
```
`profile` values (integer): `0` balanced, `1` power_save, `2` performance

#### `UpdateProgress`
```json
{ "v": 1, "type": "UpdateProgress", "step": "verify", "percent": 25 }
```
`step` values: `verify`, `write_inactive_slot`, `write_efi`, `update_boot_json`, `sync`. `percent` is 0–100.

#### `UpdateComplete`
```json
{ "v": 1, "type": "UpdateComplete", "active_slot": "b", "version": "0.2.0" }
```
Emitted after the inactive slot is written and `boot.json` is switched. The system requires a reboot to boot the new slot.

#### `UpdateError`
```json
{ "v": 1, "type": "UpdateError", "step": "verify", "reason": "signature_invalid" }
```
Emitted when an update fails after being accepted. `reason` matches the `ApplyUpdateError` reason set.

---

## 6. Lifecycle Transport — per-game fd

`playos-init` passes `PLAYOS_LIFECYCLE_FD=<n>` in the game's environment. The fd is the read end of a pipe.

Each event is a **single byte**:

| Byte value | Event |
|---|---|
| `0x00` | `PLAYOS_LIFECYCLE_FOREGROUND` |
| `0x01` | `PLAYOS_LIFECYCLE_BACKGROUND` |
| `0x02` | `PLAYOS_LIFECYCLE_SUSPEND` |
| `0x03` | `PLAYOS_LIFECYCLE_RESUME` |
| `0x04` | `PLAYOS_LIFECYCLE_TERMINATE` |

On `EOF` (pipe write end closed): treated as `TERMINATE`.

`playos_lifecycle_poll()` in `libplayos` reads from this fd.

---

## 7. Compositor Control Channel

`playos-init` (and `playos-runtime` client library) communicates with `playos-compositor` via `/run/playos/compositor.sock`.

This channel uses the same framing as control IPC.

### `SetExpectedGame`
```json
{ "v": 1, "type": "SetExpectedGame", "launch_token": "<uuid>", "game_id": "com.example.game" }
```
Tells the compositor which Wayland client to expect. The compositor matches by checking the `PLAYOS_LAUNCH_TOKEN` environment variable of connecting clients.

### `ClearExpectedGame`
```json
{ "v": 1, "type": "ClearExpectedGame" }
```

### `ForceTerminateGame`
```json
{ "v": 1, "type": "ForceTerminateGame" }
```
Compositor destroys the game surface immediately (for crash recovery). `playos-init` handles the actual process kill.

### `ShowOverlay`
```json
{ "v": 1, "type": "ShowOverlay" }
```

### `HideOverlay`
```json
{ "v": 1, "type": "HideOverlay" }
```

### Compositor → init events

#### `GameSurfaceReady`
```json
{ "v": 1, "type": "GameSurfaceReady", "launch_token": "<uuid>" }
```
Emitted when the game commits its first valid buffer. `playos-init` records this as a successful launch.

#### `CompositorStateChanged`
```json
{ "v": 1, "type": "CompositorStateChanged", "state": "GAME_FOREGROUND" }
```
`state` values: `SHELL_FOREGROUND`, `GAME_STARTING`, `GAME_FOREGROUND`, `PLAYOS_UI_FOREGROUND_WITH_GAME_BACKGROUND`, `TERMINATING_GAME`

---

## 8. Protocol Versioning

All messages include `"v": <version_integer>`. The current version is `1`.

**On version mismatch:**
```json
{ "v": 1, "type": "ProtocolError", "reason": "version_mismatch", "supported": [1] }
```
The receiver closes the connection after sending this error.

**Backward compatibility:** A server implementing version N must also accept messages with `"v": M` where M < N, treating unknown fields as ignored. It must not accept `"v": M` where M > N.

---

## 9. Error Handling

All request messages may receive a generic error response:

```json
{ "v": 1, "type": "Error", "reason": "internal_error", "message": "..." }
```

`reason` values: `version_mismatch`, `invalid_message`, `permission_denied`, `internal_error`, `not_implemented`

**Connection loss:** If `playos-init` loses a trusted client connection unexpectedly, it logs the event. This does not affect system operation. Clients should reconnect with exponential backoff.

**Rate limiting:** `playos-init` may reject rapid repeated requests (e.g., rapid `LaunchGame` calls) with `reason: "rate_limited"`.

---

## Network control (Sprint 16)

Wi-Fi rides the same control plane as everything else: the shell sends a request
to `control.sock`, `playos-init` relays it to the trusted **`playos-net`** bridge
(`/run/playos/net/bridge.sock`, `root:playos-trusted` 0660), and the bridge
answers on the same connection. The shell never talks to wpa_supplicant, and
games are not in `playos-trusted`, so they cannot reach either socket.

`playos-net` is the only process that speaks wpa_supplicant's control protocol.
It discovers the wireless interface from `/sys/class/net/*/wireless` — the Ally's
is **`wlp6s0`** (predictable naming), not `wlan0` — and wpa_supplicant's own
control socket is `/run/playos/net/<ifname>`.

| Direction | Message | Fields |
|---|---|---|
| shell → init → net | `ScanNetworks` | — |
| net → shell | `ScanResults` | `networks[]` of `{ssid, security, signal_dbm}` |
| shell → init → net | `ConnectNetwork` | `ssid`, `psk`, `security` (`open`\|`wpa2`\|`wpa3`) |
| net → shell | `ConnectNetworkAck` | `ssid` |
| net → shell | `ConnectNetworkError` | `ssid`, `reason` (`auth_failed`, `no_wpa`, …) |
| shell → init → net | `DisconnectNetwork` | — |
| shell → init → net | `NetworkStatus` | — |
| net → shell | `NetworkStatusReport` | `state`, `ssid`, `ip`, `signal_dbm` |
| net → shell (async) | `NetworkStateChanged` | `state` (`connecting`\|`connected`\|`disconnected`) |

`state` is derived from wpa_supplicant's `wpa_state` (`COMPLETED` → connected;
`SCANNING`/`ASSOCIATING`/`4WAY_HANDSHAKE`/… → connecting; otherwise
disconnected). Known networks persist as `/data/config/network/<slug>.json`
(0600 — the PSK is never logged) with a `last` pointer that `playos-net` uses to
auto-connect on boot.

Verified on the ROG Ally against the real radio:

    → {"v":1,"type":"ScanNetworks"}
    ← {"v":1,"type":"ScanResults","networks":[
         {"ssid":"messaritisnikhouse","security":"wpa2","signal_dbm":-25},
         {"ssid":"ARRIS-2468_EXT","security":"wpa2","signal_dbm":-77}, … ]}

---

## Text entry / OSK (designed 2026-09-26)

The OSK is a system service, not a shell widget: any **foreground** client can ask for text, and the
keyboard itself is rendered by `playos-overlay`, so an untrusted game cannot present a fake one
(ADR-0013). This replaces the earlier plan of `zwp_text_input_v3` and compositor-driven visibility,
which assumed Wayland surfaces that first-party clients do not have.

**Flow.** requester → compositor (broker) → overlay renders the keyboard and owns the edit buffer →
compositor → requester.

**Transport (corrected 2026-09-26 after reading the code).** The first draft said games would
request over their lifecycle fd. They cannot: that fd is a one-byte, init-to-game channel
(`FOREGROUND/BACKGROUND/SUSPEND/RESUME/TERMINATE`) with no way back. A game's only outbound channel
is its Wayland connection, carrying the **`playos-v1`** PlayOS protocol - which is exactly the
"explicit PlayOS API" ADR-0013 calls for, and is our own protocol rather than a foreign one.

So the two paths converge on the **compositor**, which owns the overlay and already tracks which
client is foreground:

- **trusted clients** (shell) -> `control.sock` -> `playos-init` -> compositor
- **games** -> `playos-v1` on their existing Wayland connection -> compositor
- **compositor** -> overlay (`ShowKeyboard`, via the existing overlay coordination) -> OSK
- text back: overlay -> compositor -> the requester, and nowhere else

`playos-init` therefore relays for trusted clients only (as it does for the network control plane);
it is not in the game path at all.

| type | direction | body |
|---|---|---|
| `RequestText` | client → init | `{prompt, max_len, masked}` |
| `TextRequestState` | init → client | `{state: "open" \| "closed"}` |
| `TextChanged` | init → client | `{text}` — the requester mirrors what is typed |
| `TextCommitted` | init → client | `{text}` |
| `TextCancelled` | init → client | `{}` |
| `ShowKeyboard` | init → compositor | `{prompt, max_len, masked}` — the compositor raises the overlay, which is how the OSK appears above a running game |

**Rules**

- **Only the foreground client may hold a request.** `playos-init` checks it against the state
  machine, so a background game cannot steal the keyboard or the focus.
- **The text goes to the requester and nowhere else.** With `masked` set the requester still receives
  the characters — it is the one who asked — but the overlay renders them masked and no other client
  sees them. That is what makes the Wi-Fi passphrase field safe to serve with the same mechanism.
- **The overlay owns the edit buffer**; the requester sees updates and the final commit, so its own
  field shows characters as they are typed.
- **Input suppression is cooperative.** Games read evdev directly (ADR-0013), so a game keeps seeing
  the d-pad while the OSK is up. The SDK reports text-entry-active and gameplay input is expected to
  be suspended; a game that ignores this only sabotages its own input handling, and nothing else
  depends on its restraint.
- **The OSK must keep the shell/overlay marked busy while visible**, since idle redraw is ~8 fps.

**`libplayos` surface (to implement):** `playos_input_request_text(const PlayOSTextRequest *)` and
`playos_input_poll_text(PlayOSTextEvent *)`, following the existing poll-based lifecycle pattern.

