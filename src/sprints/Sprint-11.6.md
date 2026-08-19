# Sprint 11.6 — Developer SSH (Dropbear) + Minimal Wired Network Bring-Up

**Goal:** Unblock Sprint 11.5's ROG Ally A/B hardware validation (and future on-device debugging) by adding a minimal, developer-only SSH path over USB-C Ethernet now, while keeping full Wi-Fi networking in Sprint 16.

**Primary Outcome:** With a USB-C Ethernet adapter plugged into the ROG Ally, the device obtains an IPv4 address via DHCP and exposes a Dropbear SSH server that only accepts developer-supplied public-key authentication. Host keys and `authorized_keys` persist under `/data/ssh`; `playos-init` supervises the bring-up; games and production behavior are unchanged.

**Status:** 🟡 Planned — not started. Scope locked as **Option 3** (USB-C Ethernet now, Wi-Fi = Sprint 16 later).

**Prerequisites:** Sprint 11.5 in progress (hardware A/B matrix pending); Sprint 10 installed/deployed rootfs path; `playos-init` supervision framework in place.

---

## Why This Sprint Exists

Sprint 11.5 needs a full 6-case A/B matrix on real ROG Ally hardware, but debugging that matrix on a handheld without a keyboard is painful — log files on the USB stick only go so far when the issue is a boot or rollback edge case. A wired SSH shell makes it practical to inspect `boot.json`, `/proc/mounts`, and live logs directly. Full Wi-Fi is a larger, later sprint (Sprint 16) with `playos-net`, `wpa_supplicant`, `dhcpcd`, and a shell UI; that work should not be pulled forward just to get a debug shell. This sprint therefore delivers the **minimal wired slice now** and leaves Wi-Fi where it is.

---

## Start Condition Checklist

- [x] Sprint 11.5 T1–T4 landed; hardware A/B matrix still pending.
- [x] `playos-init` has a supervision loop that forks/execs trusted daemons after the data mount.
- [x] `/data` is the persistent writable partition (label `playos-data`), mounted read-write by `playos_mount_data`.
- [x] Rootfs is read-only squashfs; persistent state must live under `/data`, not `/etc`.
- [x] `playos-refdistro` Ally defconfigs (`playos_ally_defconfig`, `playos_ally_installer_defconfig`) currently enable neither Dropbear nor network packages.
- [x] Ally kernel configs defer `CONFIG_NETDEVICES` (`# CONFIG_NETDEVICES is not set`) and `# CONFIG_WIRELESS is not set`.
- [x] Buildroot has Dropbear 2026.94, `wpa_supplicant`, `dhcpcd`, `connman`, and `network-manager` available; this sprint uses **Dropbear + `dhcpcd` only**.

---

## Decisions Locked for This Sprint

- **Option 3 — Both, staged.** Deliver minimal wired SSH (USB-C Ethernet) now; keep full Wi-Fi in Sprint 16. Sprint 16 is **not** absorbed or renamed by this sprint.
- **Transport now:** USB-C Ethernet via common USB-NIC drivers (ASIX `AX8817x`/`ASIX`, Realtek `RTL8152`, CDC Ethernet/NCM/EEM, RNDIS host, SMSC95xx/MCS7830). No Wi-Fi, no `RFKILL`, no `nl80211`, no `wpa_supplicant` in this sprint.
- **SSH daemon:** Dropbear (`BR2_PACKAGE_DROPBEAR=y`), **public-key auth only**. No password auth, no baked credentials, no committed private keys. Disable reverse DNS (`BR2_PACKAGE_DROPBEAR_DISABLE_REVERSEDNS=y`) and include the client (`BR2_PACKAGE_DROPBEAR_CLIENT=y`).
- **DHCP client:** standalone `dhcpcd` (`BR2_PACKAGE_DHCPCD`), exactly as Sprint 16 already locks (Option B, `network-options.md` §10). No BusyBox `udhcpc`/`ip` networking applets are added, so the wired debug path is consistent with the future Wi-Fi path and avoids re-introducing a BusyBox dependency Sprint 12/16 must later remove.
- **Persistent SSH state:** Dropbear host keys and `authorized_keys` live under `/data/ssh/` (persistent `playos-data`), **not** `/etc/dropbear` or `/var/run/dropbear` (tmpfs/ephemeral) and **not** on the read-only squashfs. Bring-up bind-mounts `/data/ssh` over `/root/.ssh` (Dropbear's default `authorized_keys` location).
- **Bring-up ownership:** a small rootfs-overlay helper (`/usr/bin/playos-ssh-bringup`) handles interface bring-up, DHCP, host-key generation, bind-mounting, and `exec dropbear`. `playos-init` forks/execs it as a supervised trusted daemon after the data mount, alongside compositor/shell/overlay.
- **Access channel:** developer-only, no shell UI, no `playos-runtime` messages, no game access. Access is gated by public-key auth: a developer drops their public key at `/data/ssh/authorized_keys`.
- **Production split:** this sprint enables SSH in the **dev/installer** image. Sprint 12 removes Dropbear and BusyBox from the production image (as already specified in S12-T7).

---

## Scope

### In Scope

- `playos-refdistro` — kernel config: enable `NETDEVICES` plus USB-NIC and QEMU test NIC drivers (no wireless).
- `playos-refdistro` — Ally/installer defconfigs: enable `dropbear` (+ client, `DISABLE_REVERSEDNS`) and `dhcpcd`.
- `playos-refdistro` — rootfs overlay: `/usr/bin/playos-ssh-bringup` and a build-time `/root/.ssh` directory (bind-mount target on the read-only squashfs).
- `playos-init` — spawn and supervise `playos-ssh-bringup` after `/data` mounts.
- `playos-spec` — this document; `SUMMARY.md` + `roadmap.md` wiring; a networking note in `kernel-config.md`.

### Explicitly Out of Scope

- Wi-Fi (`CFG80211`/`MAC80211`/`MT7921E`, `wpa_supplicant`, `dhcpcd`, `playos-net`, shell Wi-Fi UI, profiles) — **Sprint 16**.
- Bluetooth, D-Bus, NetworkManager, `connman`, `iwd`, `openssh`.
- Game/application network access and per-game network allowlists.
- On-device SSH management UI, profiles, or key-management screens.
- OTA download and `playos-tools` networking — still post-MVP / after Sprint 16.
- Production hardening: removing Dropbear/BusyBox from the production image is Sprint 12, not this sprint.

---

## Required Repository Changes

| Repo | Required work |
|---|---|
| `playos-refdistro` | Kernel USB-NIC config; Dropbear + `dhcpcd` packages; `playos-ssh-bringup` overlay script; `/root/.ssh` build-time directory |
| `playos-init` | Supervise `playos-ssh-bringup` as a trusted daemon after the data mount |
| `playos-spec` | This sprint; `SUMMARY.md` and `roadmap.md` links; `kernel-config.md` networking note |

---

## Expected Files and Directories

### `playos-refdistro`

```text
br2-external/board/ally/linux.config                 # enable NETDEVICES + USB-NIC drivers
br2-external/board/ally/linux-installer.config       # same for installer kernel
br2-external/configs/playos_ally_defconfig           # dropbear + dhcpcd
br2-external/configs/playos_ally_installer_defconfig # dropbear + dhcpcd
br2-external/board/common/rootfs-overlay/usr/bin/playos-ssh-bringup
br2-external/board/common/rootfs-overlay/root/.ssh/  # bind-mount target (exists in squashfs)
```

### `playos-init`

```text
src/supervisor.c                # spawn_ssh_bringup() alongside the other trusted daemons
include/playos-init/supervisor.h # declaration (path verified against existing headers)
```

### `playos-spec`

```text
src/sprints/Sprint-11.6.md      # this document
src/kernel-config.md            # note: USB-NIC enabled for developer SSH; Wi-Fi still deferred
```

---

## Agent Task Breakdown

### Task Status Grid

| Task ID | Task | Primary repo | Status | Notes / evidence |
|---|---|---|---|---|
| S11.6-T1 | Enable USB-NIC kernel config (no wireless) | `playos-refdistro` | not started | Verify exact Kconfig names against the 6.12 tree before editing |
| S11.6-T2 | Enable Dropbear + `dhcpcd` | `playos-refdistro` | not started | Key auth only; `DISABLE_REVERSEDNS` |
| S11.6-T3 | `playos-ssh-bringup` + `playos-init` supervision | `playos-refdistro`, `playos-init` | not started | Spawn after data mount |
| S11.6-T4 | Host keys + `authorized_keys` persistence under `/data/ssh` | `playos-refdistro` | not started | Bind-mount over `/root/.ssh` |
| S11.6-T5 | QEMU + Ally validation | `playos-refdistro` | not started | interface appears, `dhcpcd` lease, SSH key login |

Update the **Status** column as work progresses: `not started` → `in progress` → `blocked` or `done`.

### S11.6-T1 — Enable USB-NIC kernel config

Candidate symbols (verify exact names against the 6.12 Kconfig tree before editing):

```kconfig
CONFIG_NETDEVICES=y
CONFIG_ETHERNET=y
CONFIG_MII=y
CONFIG_USB_NET_DRIVERS=y
CONFIG_USB_NET_AX8817X=y
CONFIG_USB_NET_ASIX=y
CONFIG_USB_NET_RTL8152=y
CONFIG_USB_NET_CDCETHER=y
CONFIG_USB_NET_CDC_NCM=y
CONFIG_USB_NET_CDC_EEM=y
CONFIG_USB_NET_RNDIS_HOST=y
CONFIG_USB_NET_SMSC95XX=y
CONFIG_USB_NET_MCS7830=y
# QEMU/dev test NICs:
CONFIG_VIRTIO_NET=y
CONFIG_E1000=y
CONFIG_E1000E=y
```

`CONFIG_NET`, `CONFIG_PACKET`, `CONFIG_UNIX`, and `CONFIG_INET` are already present. Do **not** enable `CONFIG_WIRELESS`, `CONFIG_CFG80211`, or `CONFIG_MAC80211` in this sprint.

**Done when:** the built kernel exposes a USB NIC (and, in QEMU, the test NIC) as an interface after module/device load.

### S11.6-T2 — Enable Dropbear + `dhcpcd`

- Add `BR2_PACKAGE_DROPBEAR=y`, `BR2_PACKAGE_DROPBEAR_CLIENT=y`, and `BR2_PACKAGE_DROPBEAR_DISABLE_REVERSEDNS=y` to the Ally and installer defconfigs.
- Add `BR2_PACKAGE_DHCPCD=y` (the same client Sprint 16 uses). No BusyBox `udhcpc`/`ip` applets are enabled.
- Keep this scoped to the **dev/installer** image; Sprint 12 strips Dropbear from production, and `dhcpcd` remains for Sprint 16.

**Done when:** `dropbear` and `dhcpcd` are present in the built rootfs and both link without errors.

### S11.6-T3 — `playos-ssh-bringup` + supervision

Create `/usr/bin/playos-ssh-bringup` in the rootfs overlay:

1. Detect a wired NIC (iterate `/sys/class/net/*`, skip loopback). If none, log and exit cleanly (USB NIC may be hot-plugged later).
2. Run `dhcpcd -q -b <if>` to bring the interface up and obtain/keep a lease (`dhcpcd` daemonises by default).
3. `mkdir -p /data/ssh` and generate Dropbear host keys with `dropbearkey` if absent (RSA/ed25519).
4. `mount --bind /data/ssh /root/.ssh` (the `/root/.ssh` target must already exist in the read-only squashfs).
5. `exec dropbear -F -R -E` (foreground so `playos-init` can supervise it; `-R`/`-E` chosen to match the configured Dropbear behavior, verified at implementation time).

`playos-init/src/supervisor.c` gains `spawn_ssh_bringup()` and calls it after `/data` mounts, alongside the other trusted daemons. Restart policy mirrors the existing supervision loop.

**Done when:** after boot, Dropbear is visible as a supervised child of PID 1, and a USB-C Ethernet adapter receives a DHCP lease.

### S11.6-T4 — Persist host keys and authorized keys

- Dropbear host keys live at `/data/ssh/dropbear_*_host_key`, generated once and reused across boots.
- `authorized_keys` lives at `/data/ssh/authorized_keys`; the bind-mount makes it visible at `/root/.ssh/authorized_keys`.
- Document the developer setup: copy the developer's public key to `/data/ssh/authorized_keys` on the mounted `playos-data` partition.
- Do **not** commit any private key or seed an `authorized_keys` in the repo.

**Done when:** a reboot preserves the host key (no re-generation / client key-change warning) and a public key in `/data/ssh/authorized_keys` permits login.

### S11.6-T5 — QEMU + Ally validation

- QEMU: boot with a NIC (e.g. `-netdev user -device virtio-net-pci`); assert a wired interface appears (via `/sys/class/net`) and `dhcpcd` obtains a lease; assert SSH key login works.
- Ally: with a USB-C Ethernet adapter, assert `ip addr` + DHCP lease and SSH key login; confirm no impact on audio/input/shell boot.
- Confirm production image is unchanged by this sprint (Sprint 12 remains responsible for stripping debug tools).

**Done when:** both QEMU and Ally show a DHCP lease and a successful public-key SSH login, with log evidence.

---

## Implementation Guidance

**Verify Kconfig symbol names first.** The Linux 6.12 tree has renamed/aliased some USB-NIC symbols; confirm the exact names against `buildroot/output/*/build/linux-*` before editing `linux.config` and `linux-installer.config`.

**Persistent state belongs on `/data`, never `/etc` or `/var/run`.** The rootfs is read-only squashfs and `/var/run` is tmpfs, so Dropbear host keys must be generated under `/data/ssh`.

**Create the bind-mount target at build time.** Because the squashfs is read-only, `mkdir -p /root/.ssh` at runtime would fail; the `/root/.ssh` directory must exist in the rootfs overlay before the image is built.

**Key auth only.** Do not enable password auth and do not bake any credential into the image. A developer enables access by placing their public key at `/data/ssh/authorized_keys`.

**Keep it developer-only.** No shell settings screen, no `playos-runtime` IPC, no game access, no production removal — production removal is Sprint 12.

---

## Verification and Evidence

| Evidence | How it is produced |
|---|---|
| USB NIC appears | `/sys/class/net` shows an `eth*`/`en*` interface |
| DHCP lease | `dhcpcd` log shows an IPv4 address assigned |
| SSH supervised | `playos-init` supervision log shows Dropbear as a child; `ps` shows it under PID 1 |
| Key persistence | reboot without host-key regeneration; existing `authorized_keys` still works |
| Login works | `ssh -i <devkey> root@<ip>` succeeds |
| No wireless pulled in | kernel config still has `# CONFIG_WIRELESS is not set` |
| Production unchanged | production defconfig still excludes Dropbear/BusyBox (Sprint 12) |

---

## Acceptance Criteria

- [ ] ROG Ally with USB-C Ethernet obtains an IPv4 address via DHCP
- [ ] Dropbear SSH accepts a developer-supplied public key and rejects password login
- [ ] Dropbear host keys persist across reboots under `/data/ssh`
- [ ] `authorized_keys` persists under `/data/ssh` and survives reboot
- [ ] `playos-init` supervises the SSH bring-up as a trusted daemon
- [ ] QEMU boot with a NIC reaches a DHCP lease and SSH login
- [ ] No Wi-Fi or `wpa_supplicant` code is introduced in this sprint
- [ ] Games and existing shell/audio/input behavior are unchanged

---

## Handoff to Sprint 12 / Sprint 16

Sprint 12 (Security Hardening) may assume:

- SSH (`dropbear`) is enabled in the **dev/installer** image and must be removed from the **production** image (S12-T7).
- The SSH bring-up is a trusted, supervised child of `playos-init`, not a user-facing feature.

Sprint 16 (`playos-net`) may assume:

- The immediate wired debug path already uses `dhcpcd`, so Sprint 16 reuses the same client for the Wi-Fi interface instead of introducing a different DHCP stack.
- Sprint 16 adds Wi-Fi (`wpa_supplicant` + `playos-net`) on top of `dhcpcd`, without absorbing this sprint.

---

## Exit Gate

A ROG Ally with a USB-C Ethernet adapter obtains a DHCP lease and accepts a public-key SSH login for on-device debugging, with host keys and `authorized_keys` persisted under `/data/ssh`, while full Wi-Fi remains deferred to Sprint 16.

*Previous: [Sprint 11.5](Sprint-11.5.md) | Next: [Sprint 12](Sprint-12.md)*
