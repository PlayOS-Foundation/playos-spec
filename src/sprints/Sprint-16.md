# Sprint 16 — `playos-net` (Wi-Fi Networking)

**Goal:** Bring up Wi-Fi on the ROG Ally with a minimal, D-Bus-free stack — `wpa_supplicant` + `dhcpcd` + a trusted `playos-net` bridge — exposed to the shell through the existing `playos-runtime` control IPC. No D-Bus, no NetworkManager, no BusyBox in production.

**Primary Outcome:** The ROG Ally scans for networks, connects to a WPA2/WPA3 network, obtains an IP via DHCP, and the shell shows a working Wi-Fi settings screen (scan → connect → connected status). The whole path is driven through `control.sock`, exactly like `LaunchGame`.

**Status:** 🟡 Post-MVP — not started. Stack decision recorded in [`network-options.md`](network-options.md) §10 (Option B).

**Prerequisites:** MVP complete (Sprint 15) and Sprint 12 security hardening (Landlock/seccomp, `playos-trusted` group) in place.

---

## Why This Sprint Exists

MVP deliberately ships with no network stack. Every Tier-1 post-MVP feature — store downloads, cloud saves, network update download, and SSH Developer Mode — depends on Wi-Fi. This sprint delivers the first networking capability while honouring the core architectural constraint: **the existing `playos-runtime` IPC is the only control plane, and D-Bus is not introduced.**

See [`network-options.md`](network-options.md) for the full options analysis (iwd + D-Bus vs wpa_supplicant vs custom nl80211).

---

## Start Condition Checklist

- Sprint 15 complete; MVP (19 criteria in `roadmap.md`) verified on hardware.
- Sprint 12 hardening merged: `playos-trusted` group, Landlock, seccomp, production image has no BusyBox.
- `network-options.md` §10 decision accepted (Option B — `wpa_supplicant` + `dhcpcd`).
- Kernel currently defers `CFG80211`/`MAC80211`/`MT7921E` (`kernel-config.md` §Networking).

---

## Decisions Locked for This Sprint

- **`wpa_supplicant`, not `iwd`** — built D-Bus-free: `CONFIG_CTRL_IFACE=unix`, `CONFIG_CTRL_IFACE_DBUS=n`. Talks to the kernel over `nl80211`.
- **`dhcpcd`, not BusyBox `udhcpc`** — standalone DHCPv4/DHCPv6 + IPv4LL client (`BR2_PACKAGE_DHCPCD`). Production has no BusyBox.
- **No D-Bus.** This is the entire point of the chosen stack.
- **`playos-net` bridge daemon** — links `libwpa_client` (`wpa_ctrl`), translates wpa_supplicant's control protocol ↔ `playos-runtime` JSON frames.
- **Control plane = `control.sock`** — new network messages ride the existing trusted socket; the shell is the only UI.
- **Trust boundary** — `wpa_supplicant` and `dhcpcd` control sockets live under `/run/playos/net/`, owned `root:playos-trusted`, mode `0660`. Games are not in `playos-trusted`, so they never reach them.
- **Auth scope** — WPA2-PSK and WPA3-SAE (wpa_supplicant's in-tree SAE). No EAP/enterprise (802.1X) yet.
- **No game network access** — networking is a system/shell capability only. A per-game allowlist is a separate, later decision.

---

## Scope

### In Scope

- Kernel: `CFG80211`, `MAC80211`, `MT7921E` (AMD RZ616 = rebranded MediaTek MT7922), `RFKILL`.
- MediaTek `mt7921`/`mt7922` firmware blobs (redistributable via `linux-firmware`).
- Buildroot packages: `wpa_supplicant` (D-Bus disabled), `dhcpcd`, `playos-net`.
- `playos-net` daemon (new): wpa_supplicant control socket ↔ `playos-runtime` IPC bridge.
- `playos-runtime`: new control messages (scan, connect, disconnect, status, async events).
- `playos-init`: spawn and supervise `wpa_supplicant`, `dhcpcd`, and `playos-net`.
- `playos-shell`: Wi-Fi settings screen (scan list, connect with passphrase, live status).
- Network profiles persisted under `/data/config/network/` (SSID + PSK).

### Explicitly Out of Scope

- Bluetooth (separate — BlueZ is D-Bus-only; see `network-options.md` §8).
- D-Bus, NetworkManager, `iwd`.
- Game/application network access (per-game allowlist is a later decision).
- EAP/enterprise auth (802.1X), VPN, proxy.
- SSH Developer Mode (Dropbear) — depends on this sprint but is its own work package.
- Wi-Fi Direct, hotspot, mesh, captive-portal detection.

---

## Required Repository Changes

| Repo | Required work |
|---|---|
| `playos-net` (new) | Bridge daemon: `wpa_ctrl` ↔ `playos-runtime` JSON; profile management |
| `playos-runtime` | Network control messages + framing docs |
| `playos-init` | Supervise `wpa_supplicant`/`dhcpcd`/`playos-net`; network policy |
| `playos-shell` | Wi-Fi settings screen |
| `playos-refdistro` | Kernel config, wpa_supplicant + dhcpcd + playos-net packages, firmware overlay |
| `playos-spec` | This sprint; `runtime-ipc.md` network messages; `kernel-config.md` networking section |

> `playos-net` is a new small daemon. It may start as `playos-refdistro/src/playos-net/` (as `playos-overlay` did) before promotion to its own repo.

---

## Expected Files and Directories

### `playos-net` (new)

```text
src/main.c                    # daemon loop: connect to wpa_ctrl + control.sock
src/wpa_bridge.c              # wpa_supplicant control-protocol translation
src/profiles.c                # load/store /data/config/network/*.json
include/playos_net.h          # internal message types (mirrors runtime IPC)
```

### `playos-runtime`

```text
proto/network.json            # new message schemas (Scan/Connect/Disconnect/Status)
```

### `playos-refdistro`

```text
br2-external/configs/playos_rog_ally_defconfig   # enable CFG80211/MAC80211/MT7921E/RFKILL
board/playos/rog-ally/rootfs-overlay/lib/firmware/mediatek/   # mt7921/mt7922 blobs
br2-external/package/playos-net/                 # new package
```

### `playos-shell`

```text
src/ui/network.c              # Wi-Fi settings screen (scan/connect/status)
```

---

## Agent Task Breakdown

### Task Status Grid

| Task ID | Task | Primary repo | Status | Notes / evidence |
|---|---|---|---|---|
| S16-T1 | Enable Wi-Fi kernel config + firmware | `playos-refdistro` | not started | `CFG80211`/`MAC80211`/`MT7921E` currently deferred |
| S16-T2 | Package wpa_supplicant (D-Bus-free) + dhcpcd | `playos-refdistro` | not started | `CONFIG_CTRL_IFACE_DBUS=n` |
| S16-T3 | Implement `playos-net` bridge daemon | `playos-net` | not started | wpa_ctrl ↔ control.sock |
| S16-T4 | Add network messages to `playos-runtime` | `playos-runtime` | not started | additive; keep `v: 1` |
| S16-T5 | Supervise network daemons in `playos-init` | `playos-init` | not started | |
| S16-T6 | Wi-Fi settings screen in `playos-shell` | `playos-shell` | not started | |
| S16-T7 | Network profile persistence | `playos-net` | not started | `/data/config/network/` |
| S16-T8 | End-to-end validation (Ally + QEMU) | `playos-refdistro` | not started | |

### S16-T1 — Enable Wi-Fi kernel config + firmware

```kconfig
CONFIG_CFG80211=y
CONFIG_MAC80211=y
CONFIG_MT7921E=y            # AMD RZ616 (MediaTek MT7922) on ROG Ally
CONFIG_RFKILL=y
```

- Add the MediaTek `mt7921`/`mt7922` Wi-Fi firmware to `board/playos/rog-ally/rootfs-overlay/lib/firmware/mediatek/` (redistributable via `linux-firmware`, unlike AMD GPU blobs).
- **Done when:** the Ally's `mt7921e` interface appears (`ip link` shows `wlan0`/`mlan0` after firmware load).

### S16-T2 — Package `wpa_supplicant` (D-Bus-free) + `dhcpcd`

- Build `wpa_supplicant` with `CONFIG_CTRL_IFACE=unix`, `CONFIG_CTRL_IFACE_DBUS=n`; internal crypto (no kernel-crypto dependency).
- Build `dhcpcd` (`BR2_PACKAGE_DHCPCD`).
- Control sockets: `/run/playos/net/wpa.sock` and `/run/playos/net/dhcpcd.sock`, owned `root:playos-trusted` `0660`.
- **Done when:** both binaries link and their control sockets are restricted to the trusted group.

### S16-T3 — Implement `playos-net` bridge daemon

- Links `libwpa_client`; connects to `/run/playos/net/wpa.sock` and `control.sock`.
- Translates wpa_supplicant control-protocol events (`CTRL-EVENT-CONNECTED`, `CTRL-EVENT-SCAN-RESULTS`, `CTRL-EVENT-DISCONNECTED`) into `playos-runtime` JSON.
- Runs with dropped privileges in `playos-trusted`; never exposes wpa_supplicant directly to games.
- **Done when:** a `ScanNetworks` request on `control.sock` returns a live scan result list.

### S16-T4 — Add network messages to `playos-runtime`

New additive messages on `control.sock` (keep `"v": 1`; framing unchanged):

```json
{ "v": 1, "type": "ScanNetworks" }
{ "v": 1, "type": "ScanResults", "networks": [ { "ssid": "…", "security": "wpa2", "signal_dbm": -54 } ] }

{ "v": 1, "type": "ConnectNetwork", "ssid": "…", "psk": "…", "security": "wpa2" }
{ "v": 1, "type": "ConnectNetworkAck", "ssid": "…" }
{ "v": 1, "type": "ConnectNetworkError", "ssid": "…", "reason": "auth_failed" }

{ "v": 1, "type": "DisconnectNetwork" }
{ "v": 1, "type": "NetworkStatus" }
{ "v": 1, "type": "NetworkStatusReport", "state": "connected", "ssid": "…", "ip": "192.168.1.10", "signal_dbm": -54 }

{ "v": 1, "type": "NetworkStateChanged", "state": "connecting" }   /* async: connecting|connected|disconnected */
```

**Done when:** the message set is documented in `runtime-ipc.md` and the schemas build.

### S16-T5 — Supervise network daemons in `playos-init`

- `playos-init` spawns and supervises `wpa_supplicant`, `dhcpcd`, and `playos-net` after the data partition mounts (network profiles live on `/data`).
- Restart policy mirrors other trusted daemons (exponential backoff, log on crash).
- **Done when:** all three daemons appear as supervised children of PID 1 and survive a `kill -9` restart.

### S16-T6 — Wi-Fi settings screen in `playos-shell`

- Scan list with SSID, signal strength, and security badge.
- Connect flow: select network → on-screen passphrase entry (overlay virtual keyboard) → connect → status.
- Live status indicator (connected SSID + IP, or "no network").
- All actions go through `control.sock`; the shell never talks wpa_supplicant directly.
- **Done when:** navigating Settings → Wi-Fi shows real networks and connects with a passphrase.

### S16-T7 — Network profile persistence

- Store known networks under `/data/config/network/<profile>.json` (SSID + PSK; never log the PSK).
- Auto-connect to the most recently used known network on boot.
- **Done when:** after a reboot, the Ally reconnects to a previously saved network without re-entering the passphrase.

### S16-T8 — End-to-end validation (Ally + QEMU)

- Real connection: scan → connect (WPA2-PSK and WPA3-SAE) → DHCP lease → reach the gateway.
- Lifecycle: airplane/off state, disconnect, reconnect, reboot persistence.
- Trust boundary: as `playos-game`, `connect()` to `/run/playos/net/wpa.sock` returns `EACCES`.
- Production lint: no D-Bus, no BusyBox, no `iwd` in the image.
- QEMU CI: kernel config builds; daemons start; scan fails gracefully (no radio) without crash.
- **Done when:** all cases pass with evidence logged.

---

## Verification and Evidence

| Evidence | How it is produced |
|---|---|
| Interface up | `ip link` shows the `mt7921e` interface |
| Successful association | `wpa_supplicant` log + `NetworkStateChanged: connected` on `control.sock` |
| DHCP lease | `NetworkStatusReport.ip` populated |
| Scan results | `ScanResults` JSON contains the test SSID |
| Trust boundary | `playos-game` connect to net sockets → `EACCES` |
| No D-Bus/BusyBox | Sprint 12 production lint passes |
| Reconnect after reboot | Profile reload → auto-connect log |

---

## Acceptance Criteria

- [ ] The ROG Ally scans and lists nearby networks in the shell
- [ ] Connecting to a WPA2-PSK network obtains a DHCP lease and reaches the gateway
- [ ] Connecting to a WPA3-SAE network works (where hardware/AP supports SAE)
- [ ] Network status (SSID, IP, signal) is shown live in the shell
- [ ] Saved networks reconnect automatically after reboot
- [ ] A game process cannot reach `/run/playos/net/` sockets (`EACCES`)
- [ ] No D-Bus, BusyBox, or `iwd` present in the production image
- [ ] All network operations flow through `control.sock` (no direct wpa_supplicant access from the shell)
- [ ] CI passes (kernel config builds; daemons start; scan fails gracefully in QEMU)

---

## Handoff to Post-MVP

After this sprint, post-MVP features may assume:

- Wi-Fi is available as a system service with a stable `playos-runtime` control surface
- Network profiles persist under `/data/config/network/`
- SSH Developer Mode (Dropbear) can be layered on top of this connectivity
- Bluetooth can reuse the private-bus decision from `network-options.md` §8 independently

---

## Exit Gate

The ROG Ally connects to Wi-Fi and reaches the network end-to-end, driven entirely through the existing `playos-runtime` control IPC, with no D-Bus and no BusyBox in the production image.

*Previous: [Sprint 15](Sprint-15.md)*
