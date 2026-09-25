# Sprint 16 — `playos-net` (Wi-Fi Networking)

**Goal:** Bring up Wi-Fi on the ROG Ally with a minimal, D-Bus-free stack — `wpa_supplicant` + `dhcpcd` + a trusted `playos-net` bridge — exposed to the shell through the existing `playos-runtime` control IPC. No D-Bus, no NetworkManager, no BusyBox in production.

**Primary Outcome:** The ROG Ally scans for networks, connects to a WPA2/WPA3 network, obtains an IP via DHCP, and the shell shows a working Wi-Fi settings screen (scan → connect → connected status). The whole path is driven through `control.sock`, exactly like `LaunchGame`.

**Status:** ✅ Done — **T1–T8 complete and verified on the ROG Ally (2026-09-24)**. Kernel + MT7922 firmware, D-Bus-free `wpa_supplicant`/`dhcpcd`, the trusted `playos-net` bridge, the IPC message set + `runtime-ipc.md`, init supervision with the `control.sock` relay, the Settings → Network screen, profile persistence and E2E validation all landed. Evidence: `ScanNetworks` through `control.sock` returned live networks; `kill -9` on the bridge → relaunch in 0.8 s; the saved profile **auto-connected with the ethernet dock unplugged** (reachable ~10 s after power-on, `192.168.0.115/24` dynamic lease); and a uid·gid 1001 probe is refused by the bridge, control and compositor sockets (EACCES). T8's **QEMU** half is verified too (no-radio path, see the T8 row).

**Prerequisites:** MVP complete (Sprint 15) and Sprint 12 security hardening (Landlock/seccomp, `playos-trusted` group) in place.

---

## Why This Sprint Exists

MVP deliberately ships with no network stack. Every Tier-1 post-MVP feature — store downloads, cloud saves, network update download, and SSH Developer Mode — depends on Wi-Fi. This sprint delivers the first networking capability while honouring the core architectural constraint: **the existing `playos-runtime` IPC is the only control plane, and D-Bus is not introduced.**

See [`network-options.md`](network-options.md) for the full options analysis (iwd + D-Bus vs wpa_supplicant vs custom nl80211).

---

## Start Condition Checklist

- Sprint 15 complete **except one parked check** — the desktop windowed run (see `Sprint-15.md` → Parked verification). Everything else in the S15 exit gate is verified: T1–T7 done, and T8's `device` + `emulator` runs pass.
- MVP verified on hardware (Sprint 14). Sprint 12 hardening merged: `playos-trusted` group, Landlock, seccomp, production image has no BusyBox.
- `network-options.md` §10 decision accepted (Option B — `wpa_supplicant` + `dhcpcd`); formalised as ADR-0012.
- ~~Kernel defers wireless entirely~~ — **done in S16-T1**: `br2-external/board/ally/linux.config` now enables `CONFIG_WIRELESS/CFG80211/MAC80211/RFKILL/WLAN/MT7921E` (`kernel-config.md` §Networking documents the same set).
- **Hardware gate:** T8's real scan/connect/DHCP/WPA3 and the trust-boundary checks need the Ally's MT7921e. Host/QEMU can complete T1–T7 and the QEMU half of T8.

---

## Decisions Locked for This Sprint

- **`wpa_supplicant`, not `iwd`** — D-Bus-free by *not selecting* Buildroot's `BR2_PACKAGE_WPA_SUPPLICANT_DBUS`; enable `BR2_PACKAGE_WPA_SUPPLICANT_NL80211`, `_CTRL_IFACE` (unix socket) and `_WPA3` (SAE). Buildroot owns the upstream `CONFIG_CTRL_IFACE*` flags; do not hand-edit them.
- **`dhcpcd`, not BusyBox `udhcpc`** — standalone DHCPv4/DHCPv6 + IPv4LL client (`BR2_PACKAGE_DHCPCD`; **already enabled** in `playos_ally_defconfig`). Production has no BusyBox.
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
| `playos-net` (new) | Bridge daemon: `wpa_ctrl` ↔ `playos-runtime` JSON; profile management. Start in-repo at `playos-refdistro/src/playos-net/` |
| `playos-runtime` | Network control messages in the trusted-control client (`src/trusted_control.c`) + the canonical IPC types (`playos-init/ipc/ipc.h`) |
| `playos-init` | Supervise `wpa_supplicant`/`dhcpcd`/`playos-net`; network policy |
| `playos-shell` | Wi-Fi UI: extend the Settings `TAB_NETWORK` + a `src/screen_network.c` (the shell has no `src/ui/` tree) |
| `playos-refdistro` | Kernel config (`board/ally/linux.config`), Buildroot packages (wpa_supplicant, playos-net), MediaTek firmware via the linux-firmware package |
| `playos-spec` | This sprint; `runtime-ipc.md` network messages; `kernel-config.md` networking section; ADR-0012 |

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
src/trusted_control.c         # ScanNetworks/ConnectNetwork/... client helpers
playos-init/ipc/ipc.h         # canonical message types (in the playos-init repo)
```

> `playos-runtime/protocols/playos-v1.xml` is the compositor's private **Wayland**
> protocol, not the runtime IPC. Network messages are JSON on `control.sock`;
> they belong with the canonical IPC types and are documented in `runtime-ipc.md`.

### `playos-refdistro`

```text
br2-external/board/ally/linux.config   # CFG80211/MAC80211/MT7921E/RFKILL (WIRELESS is off today)
br2-external/configs/playos_ally_defconfig   # wpa_supplicant options; dhcpcd already on
br2-external/package/playos-net/       # new package
```

MediaTek Wi-Fi firmware comes from Buildroot's `linux-firmware` package
(`BR2_PACKAGE_LINUX_FIRMWARE_MEDIATEK_MT7921` / `_MT7922`) — not an overlay blob.

### `playos-shell`

```text
src/screen_settings.c         # TAB_NETWORK (exists as a 4-line placeholder today)
src/screen_network.c          # scan list / connect / live status, screen_*.c conventions
```

---

## Agent Task Breakdown

### Task Status Grid

| Task ID | Task | Primary repo | Status | Notes / evidence |
|---|---|---|---|---|
| S16-T1 | Enable Wi-Fi kernel config + firmware | `playos-refdistro` | **done** | `WIRELESS/CFG80211/MAC80211/RFKILL/WLAN/MT7921E=y` in `board/ally/linux.config`; `MT7921E` selects `MT76_CORE`+`MT7921_COMMON` (checked with `olddefconfig`). Firmware from `BR2_PACKAGE_LINUX_FIRMWARE_MEDIATEK_MT7922` — the Ally's internal AMD RZ616 = MT7922, `mt7921e` driver. Verified in the shipped `rootfs.squashfs` and **on the Ally after a fresh install**: `mt7921e 0000:06:00.0: ASIC revision: 79220010`, `WM Firmware Version` logged, interface **`wlp6s0`** present with a `wireless/` dir, `rfkill0 phy0 wlan soft=0 hard=0` (unblocked). Note: the interface is **`wlp6s0`**, not `wlan0`/`mlan0` — anything downstream must discover it, not hardcode |
| S16-T2 | Package wpa_supplicant (D-Bus-free) + dhcpcd | `playos-refdistro` | **done** | `BR2_PACKAGE_WPA_SUPPLICANT_{NL80211,CTRL_IFACE,WPA3,WPA_CLIENT_SO}=y` with `_DBUS` unset, plus `BR2_PACKAGE_WIRELESS_REGDB`. Verified in the shipped image: `usr/sbin/wpa_supplicant`, `usr/lib/libwpa_client.so`, `lib/firmware/regulatory.db(.p7s)`, `etc/wpa_supplicant.conf`; **0 dbus entries**. Note `_WPA3=y` selects OpenSSL (~3 MB, ~10 min build). Socket hardening under `/run/playos/net/` lands with T3 |
| S16-T3 | Implement `playos-net` bridge daemon | `playos-net` | **done** | `src/playos-net/` (in-repo, like `playos-overlay`): `main.c` (listener + poll loop), `wpa_bridge.c` (wpa_ctrl: SCAN/STATUS/ADD_NETWORK/SELECT_NETWORK, `CTRL-EVENT-*` → `NetworkStateChanged`), `profiles.c`. Reuses `ipc_framing/ipc_client/ipc_server`; socket `/run/playos/net/bridge.sock` `root:playos-trusted` 0660 with `SO_PEERCRED` peer checks (root or GID 1000). Interface discovered from `/sys/class/net/*/wireless`. **Verified on the Ally: `ScanNetworks` → 10 live networks** |
| S16-T4 | Add network messages to `playos-runtime` | `playos-runtime` | **done** | Message strings in `playos-init/ipc/ipc.h` (`ScanNetworks`, `ScanResults`, `ConnectNetwork(Ack\|Error)`, `DisconnectNetwork`, `NetworkStatus(Report)`, `NetworkStateChanged`), documented in `runtime-ipc.md` §Network control. Client helpers land with the shell screen (T6) since that is the only caller |
| S16-T5 | Supervise network daemons in `playos-init` | `playos-init` | **done** | Supervised in init + the `control.sock` ↔ `bridge.sock` relay. Verified across two reboots (the interface-appears-late retry) and `kill -9` → relaunch in 0.8 s |
| S16-T6 | Wi-Fi settings screen in `playos-shell` | `playos-shell` | **done** | `src/screen_network.c` (scan list with signal bars + security, passphrase keyboard with CAPS / 123-symbols / SHOW-HIDE) wired into `TAB_NETWORK`; layout verified against device screenshots |
| S16-T7 | Network profile persistence | `playos-net` | **done** | `/data/config/network/<slug>.json` (0600) + `last` pointer. **Auto-connect verified with the dock unplugged** — reachable ~10 s after power-on |
| S16-T8 | End-to-end validation (Ally + QEMU) | `playos-refdistro` | **done** | Ally: scan (23 networks), connect, DHCP lease, live status; trust boundary (bridge/control/compositor sockets all refuse uid·gid 1001). QEMU (dev-mode boot, no wireless NIC, playos-data attached): init retries at the designed cadence (attempt 1, then every 10th), gives up cleanly at 60 `- no wireless interface after 60 tries - network stack not started`, and **never spawns wpa_supplicant/dhcpcd/playos-net** without a radio. Without a data disk the same image takes the install/live early-out instead (‘network stack not started (install/recovery)’) |

### S16-T1 — Enable Wi-Fi kernel config + firmware

Add to `br2-external/board/ally/linux.config` (which today says
`# CONFIG_WIRELESS is not set`):

```kconfig
CONFIG_CFG80211=y
CONFIG_MAC80211=y
CONFIG_MT7921E=y            # AMD RZ616 (MediaTek MT7922) on ROG Ally
CONFIG_RFKILL=y
```

- Enable `BR2_PACKAGE_LINUX_FIRMWARE_MEDIATEK_MT7922`. The Ally's **internal** Wi-Fi module is the AMD RZ616, a rebranded **MediaTek MT7922** (PCI `14c3:0616`, Wi-Fi 6E + BT 5.2, soldered), driven by `mt7921e`. `_MT7922` installs `WIFI_MT7922_patch_mcu_1_1_hdr.bin` + `WIFI_RAM_CODE_MT7922_1.bin`, which is exactly what the driver requests for `0616`.
  - `_MT7921` is a **sibling chip**, not the Ally's — not required (the `mt7921e` driver covers both, but the firmware differs).
  - `_MT7922_BT` / `_MT7921_BT` are **Bluetooth-only** firmware; Bluetooth is out of scope for this sprint.
  - `BR2_PACKAGE_LINUX_FIRMWARE` is already enabled. No overlay blobs — this firmware is redistributable.
- **Done when:** the Ally's `mt7921e` interface appears after firmware load.
  **Verified 2026-09-24** on the installed system: the interface is **`wlp6s0`**
  (predictable-name rename of `wlan0`), MAC `00:41:0e:f6:33:23`, `operstate=down`,
  `rfkill` unblocked. `wlan0`/`mlan0` from earlier drafts do not exist — the
  interface name comes from `mt7921e 0000:06:00.0` plus systemd/udev naming, so
  `wpa_supplicant`, `dhcpcd` and `playos-net` must take it as a parameter
  (discover by `wireless/` in `/sys/class/net/*/`), never hardcode it.
- **Buildroot trap (cost us a build):** `linux-firmware` bakes the selected files into
  `br-firmware.tar` at **build** time and the install step only extracts it, so
  enabling `_MT7922` and re-running the image build produced an image *without* the
  firmware, and `linux-firmware-reinstall` re-extracted the stale tarball.
  Use `make linux-firmware-rebuild` (or `-dirclean`) and verify the files inside
  the produced `rootfs.squashfs` before flashing.

### S16-T2 — Package `wpa_supplicant` (D-Bus-free) + `dhcpcd`

- In `playos_ally_defconfig`: `BR2_PACKAGE_WPA_SUPPLICANT=y` with `_NL80211=y`,
  `_CTRL_IFACE=y`, `_WPA3=y` and `_WPA_CLIENT_SO=y` (the bridge links
  `libwpa_client`). **Leave `BR2_PACKAGE_WPA_SUPPLICANT_DBUS` unset** — that is
  what keeps D-Bus out; Buildroot generates the upstream `CONFIG_CTRL_IFACE_*`
  flags from these options.
- `dhcpcd` (`BR2_PACKAGE_DHCPCD`) is already enabled.
- Control sockets live under `/run/playos/net/` (`root:playos-trusted`, `0770`
  for the directory). wpa_supplicant's own socket is **`/run/playos/net/<ifname>`**
  — on the Ally that is `/run/playos/net/wlp6s0`, verified on hardware, not
  `wpa.sock`; the bridge daemon serves the control plane on
  `/run/playos/net/bridge.sock`.
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

> **Hardware gate:** the real connection and trust-boundary checks below need
> the ROG Ally's MT7921e radio. Host/QEMU can only prove the kernel build, that
> the daemons start, and that a scan fails gracefully without a radio — mark
> the on-device checks separately, as Sprints 11.5–13 did.

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

## Realignment notes (2026-09-22 review)

The sprint was authored before Sprints 13.7/14/14.5/15 changed the distro and
the repos. This review corrected the spec against the current tree; **no
implementation was started.**

| Original assumption | Reality (verified) | Fix |
|---|---|---|
| `playos_rog_ally_defconfig` | `playos_ally_defconfig` | renamed |
| `board/playos/rog-ally/rootfs-overlay/lib/firmware/mediatek/` blobs | Buildroot ships `BR2_PACKAGE_LINUX_FIRMWARE_MEDIATEK_MT7922` (the Ally's internal RZ616 = MT7922) and `_MT7921` (sibling chip); `BR2_PACKAGE_LINUX_FIRMWARE` already on | firmware via the package, no overlay; **only `_MT7922` is required** |
| `wpa_supplicant` hand-tuned `CONFIG_CTRL_IFACE_DBUS=n` | Buildroot exposes `BR2_PACKAGE_WPA_SUPPLICANT_{NL80211,CTRL_IFACE,WPA3,WPA_CLIENT_SO,DBUS}` | select the options, leave `_DBUS` off |
| `dhcpcd` to be added | already `BR2_PACKAGE_DHCPCD=y` in `playos_ally_defconfig` | T2 is partly pre-done |
| `playos-runtime/proto/network.json` | repo dir is `protocols/`, and `playos-v1.xml` is the compositor's Wayland protocol; the runtime IPC is JSON in `playos-init/ipc/ipc.h` + `runtime-ipc.md` | schema pointer corrected |
| `playos-shell/src/ui/network.c` | shell has no `src/ui/`; `screen_settings.c` already has a `TAB_NETWORK` placeholder | extend the tab + add `src/screen_network.c` |
| ADR suggested as "ADR-0009" | ADR-0009 is the gamepad database; ADRs run through 0011 | Wi-Fi stack authored as **ADR-0012** |

Ground truth checked: `board/ally/linux.config` has `# CONFIG_WIRELESS is not
set` (T1 is real work); no `playos-net` repo exists (start in
`playos-refdistro/src/playos-net/`); Sprint 12's `playos-trusted` group and
hardening are in place.

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

*Previous: [Sprint 15](Sprint-15.md) | Next: [Sprint 17](Sprint-17.md)*

---

## Parked (revisit later)

One item, parked deliberately rather than left implicit.

**Wi-Fi passphrase hardening (secrets at rest).** `playos-net` persists known
networks as `/data/config/network/<slug>.json`, mode `0600 root:root`, with the
PSK in plaintext — that is what `wpa_supplicant` needs, and what lets the profile
auto-connect with no human present (T7). The posture as **verified on the Ally**:

- a game cannot read it: the file is `0600 root`, and a `uid=gid=1001` probe is
  refused by every trusted socket
- it is never logged — no `psk` appears anywhere under `/data/log/`
- it is not copied into `/run/playos/net/wpa.conf`, which holds only
  `ctrl_interface`, `ctrl_interface_group`, `ap_scan` and `update_config`
- **but** `/data` is an unencrypted ext4 partition, so the passphrase is
  readable by anyone who has the disk or mounts it on another machine

Hardening options, cheapest first: encrypt the profile with a per-device key
already present in `boot.json`; seal it with the fTPM via `/dev/tpmrm0`; or
prompt per boot and store nothing — which T7's auto-connect requirement rules
out for a console device. Not done here because it is a cross-cutting
secrets-at-rest decision affecting anything else that stores credentials, not a
Wi-Fi feature. Tracked in [`security-model.md`](../security-model.md) §12.
