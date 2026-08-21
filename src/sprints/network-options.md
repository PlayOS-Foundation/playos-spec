# Networking Options — Wi-Fi & Bluetooth

> **Status:** Design analysis (pre-ADR). Networking (Wi-Fi) is scoped as [Sprint 16](Sprint-16.md) (post-MVP); Bluetooth remains post-MVP with no sprint.
> **Cross-references:** [roadmap.md](roadmap.md) §Post-MVP, [post-mvp.md](../post-mvp.md), [architecture.md](../architecture.md) §14, [runtime-ipc.md](../runtime-ipc.md), [kernel-config.md](../kernel-config.md), [security-model.md](../security-model.md) §11

---

## 1. Context

PlayOS deliberately ships with no network stack in MVP:

- `kernel-config.md` defers `CFG80211`/`MAC80211`/`MT7921E` and the Bluetooth subsystem.
- `architecture.md` §14 lists "Wi-Fi, Bluetooth, SSH, cloud saves" as post-MVP.
- The production image contains **no BusyBox** (dev/diagnostic only — `security-model.md` §11), **no D-Bus**, and no open TCP/UDP sockets (Sprint 12). This also means production has no `/bin/sh`, so any future SSH Developer Mode (Dropbear) needs a standalone-shell vs forced-command decision for its login shell — see `post-mvp.md` §SSH Developer Mode.

The central question is therefore not *which* Wi-Fi daemon to use, but whether introducing D-Bus is acceptable. Both `iwd` and BlueZ hard-depend on D-Bus, while PlayOS's architecture mandates a strict one-mechanism-per-layer IPC model (see §3).

---

## 2. Options Summary

| Option | Stack | D-Bus? | BusyBox? | Effort | Fit |
|---|---|---|---|---|---|
| A | `iwd` + private `dbus-broker` | Yes (private bus) | No | Medium | Fits only as a contained private bus |
| B | `wpa_supplicant` + `dhcpcd` + `playos-net` bridge | **No** | No | Medium | Best architectural fit |
| C | Custom `nl80211` supplicant | No | No | Very high | Not viable |

---

## 3. The D-Bus Problem

PlayOS owns IPC by layer:

- `playos-runtime` owns **all internal IPC** — `control.sock`, `compositor.sock`, and the lifecycle fd (length-prefixed JSON frames over Unix sockets).
- `libplayos` (`playos-platform-api`) owns the **only public ABI**.
- Games are never in `playos-trusted` and cannot reach any privileged endpoint.

A system-wide D-Bus bus is a **second, parallel internal IPC mechanism**, which violates the rule that "private IPC definitions live only in `playos-runtime`" and sits alongside the explicit non-goals (`systemd`, desktop machinery). D-Bus is also a large, historically attack-surface-heavy component in a system that prioritises a small trusted surface.

**Conclusion:** D-Bus is unacceptable as a general-purpose or game-visible bus. It is tolerable **only** as a private, contained implementation detail between trusted daemons — the same trust boundary as the existing control sockets.

---

## 4. Reusing the Existing IPC as the Control Plane

A common question: PlayOS already has an IPC mechanism (`control.sock` / `compositor.sock`) — can it be used for networking instead of introducing anything new?

**Yes — but transport and implementation are distinct:**

| Concern | Owner |
|---|---|
| Carry messages between trusted components | `playos-runtime` (`control.sock`) — already exists |
| Actually do Wi-Fi (scan, associate, WPA2/WPA3 4-way handshake over `nl80211`) | `wpa_supplicant` (or `iwd`) — a daemon |

These are orthogonal. `playos-runtime` cannot speak `nl80211` without reimplementing a supplicant (Option C), so a supplicant daemon is still required underneath. The IPC mechanism is the **control plane**; the supplicant is the **engine**.

**The bridge/glue:** `wpa_supplicant` has its own control protocol over its own private socket (`libwpa_client`/`wpa_ctrl`). Something must translate between that and PlayOS's JSON frames. The glue lives in one of two places:

1. **A dedicated `playos-net` daemon** (recommended) — links `libwpa_client`, exposes new `playos-runtime` messages (`Scan`, `Connect`, `Status`) on `control.sock`. Matches the `playos-net` naming in `post-mvp.md` and keeps `playos-init` scoped to supervision (`architecture.md` already says init "Does NOT own: network").
2. **Folded into `playos-init`** — one fewer daemon, but muddies init's supervision charter.

**Result with `wpa_supplicant` (Option B):** D-Bus disappears entirely. The only sockets are the existing `control.sock`/`compositor.sock` plus `wpa_supplicant`'s and `dhcpcd`'s private sockets — all `root:playos-trusted` `0660`, invisible to games. The shell already sends `LaunchGame` over `control.sock`; it would send `Connect` the same way.

---

## 5. Option A — `iwd` + private D-Bus

`iwd` is modern, minimal, has cleaner WPA3 handling, and bundles its own DHCP client (so it needs no separate client and no BusyBox). But it is **D-Bus-only** — there is no non-D-Bus build.

To make it fit, D-Bus would be scoped as follows:

- Run `dbus-broker` (systemd-independent, minimal) on a **private socket**, owned `root:playos-trusted`, mode `0660`.
- `iwd` runs as a trusted daemon. Games are not in `playos-trusted`, so they never see the bus.
- The shell talks to `iwd` through new `playos-runtime` control messages (e.g. `Scan`, `Connect`, `Status`), not raw D-Bus.

| Pros | Cons |
|---|---|
| Modern, minimal, fast roaming | Introduces D-Bus at all |
| Bundled DHCP client | Second internal IPC mechanism (even if private) |
| Better WPA3 (SAE) support | More moving parts (`dbus-broker` + policy) |

---

## 6. Option B — `wpa_supplicant` + `dhcpcd` (D-Bus-free) — recommended

`wpa_supplicant` talks to the kernel over `nl80211` and exposes a **plain Unix socket control interface**. It can be built with D-Bus entirely disabled:

```sh
CONFIG_CTRL_IFACE=unix      # Unix socket control interface
CONFIG_CTRL_IFACE_DBUS=n    # no D-Bus dependency
```

The stack:

1. **`wpa_supplicant`** — association, WPA2/WPA3 (SAE via the in-tree hostapd/wpa_supplicant code), over `nl80211`. No D-Bus.
2. **`dhcpcd`** — standalone DHCPv4/DHCPv6 + IPv4LL client (Buildroot `BR2_PACKAGE_DHCPCD`). No BusyBox, no D-Bus.
3. **`playos-net` bridge** — a thin trusted daemon that reads wpa_supplicant's control socket and re-exposes it as new `playos-runtime` control messages, so the shell never talks to wpa_supplicant directly and games stay isolated.

The wpa_supplicant control socket becomes just another `root:playos-trusted` `0660` socket under `/run/playos/`, mirroring `control.sock` and `compositor.sock`.

| Pros | Cons |
|---|---|
| **Zero D-Bus** — matches PlayOS philosophy | Older, more config-heavy than `iwd` |
| Same Unix-socket transport as `playos-runtime` | Needs a separate DHCP client (`dhcpcd`) |
| Battle-tested in minimal embedded (OpenWrt, Alpine, Buildroot) | SAE/WPA3 support is present but less polished than `iwd` |

---

## 7. Option C — Custom `nl80211` supplicant (rejected)

Writing a minimal Wi-Fi manager that talks `nl80211` directly avoids all daemons, but re-implements the WPA2/WPA3 4-way handshake, EAP, and crypto. This is weeks of correctness- and security-critical work for zero user-visible benefit. Rejected.

---

## 8. Bluetooth

Bluetooth is the harder case: **BlueZ is D-Bus-only, and there is no D-Bus-free alternative** for modern controller/audio use.

- Basic HID pairing (controllers, keyboards) still requires BlueZ → D-Bus.
- Bluetooth audio requires the post-MVP dedicated audio service (`post-mvp.md` Tier 2), so BT is correctly deferred regardless.

Two paths:

| Path | Implication |
|---|---|
| Land Wi-Fi D-Bus-free now (Option B); introduce a private `dbus-broker` later when BT lands | Wi-Fi ships without D-Bus; BT justifies the single private bus later |
| Accept private D-Bus up front (Option A) for both | One IPC addition amortised across Wi-Fi + BT, at the cost of D-Bus presence earlier |

Either way, the D-Bus bus — if introduced for BlueZ — must be scoped exactly as in §5: private, `root:playos-trusted`, invisible to games.

---

## 9. Kernel & Firmware

```kconfig
# Wi-Fi (post-MVP enablement)
CONFIG_CFG80211=y
CONFIG_MAC80211=y
CONFIG_MT7921E=y            # AMD RZ616 (rebranded MediaTek MT7922) on ROG Ally
CONFIG_RFKILL=y

# Bluetooth (post-MVP enablement)
CONFIG_BT=y
CONFIG_BT_LE=y
CONFIG_BT_HCIBTUSB=y        # MT7922 exposes BT over USB
CONFIG_BT_RFCOMM=y          # HID profile
```

**Firmware:** MediaTek `mt7921`/`mt7922` Wi-Fi and Bluetooth blobs are **redistributable** via `linux-firmware` — unlike AMD GPU blobs, no manual sourcing from an existing install is required.

---

## 10. Recommendation

- **Wi-Fi:** Option B — `wpa_supplicant` (D-Bus-free) + `dhcpcd` + a trusted `playos-net` bridge. It lands networking with zero D-Bus and zero BusyBox, fully consistent with the existing IPC model.
- **Bluetooth:** defer. When it lands, introduce a private `dbus-broker` scoped to the trusted zone for BlueZ — the one subsystem that genuinely requires D-Bus.

**Decision:** Option B — `wpa_supplicant` (D-Bus-free) + `dhcpcd` + a trusted `playos-net` bridge — is the chosen Wi-Fi stack, decomposed as [Sprint 16](Sprint-16.md). A formal ADR is still recommended (e.g. "ADR-0009 — Wi-Fi stack: wpa_supplicant over iwd to avoid D-Bus").

**Work package:** the chosen stack is decomposed as **[Sprint 16 — `playos-net`](Sprint-16.md)**.
