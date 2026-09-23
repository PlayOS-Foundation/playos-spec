# ADR-0012 — Wi-Fi Stack: `wpa_supplicant` + `dhcpcd` (No D-Bus)

**Date:** Sprint 16 (2026-09-22, authored during the pre-implementation review)
**Status:** Accepted
**Deciders:** PlayOS core team

---

## Context

The MVP ships with **no network stack at all**. Every Tier-1 post-MVP feature —
store downloads, cloud saves, network update download, and SSH Developer Mode —
needs Wi-Fi. The ROG Ally's radio is the AMD RZ616, a rebranded **MediaTek
MT7922** (`mt7921e` driver), which is currently disabled in the Ally kernel
config.

The architectural constraint that drives this decision: existing PlayOS control
planes are **Unix sockets with a framed JSON protocol** (`control.sock`), and the
trusted zone (init, compositor, shell, overlay) is deliberately small. D-Bus is
not present, and introducing it system-wide would add a bus daemon plus policy
machinery to the trusted zone.

The options were analysed in [`../sprints/network-options.md`](../sprints/network-options.md)
§§2–10:

1. **`iwd` + D-Bus** — modern, but D-Bus is mandatory; it would introduce the bus
   into the trusted zone for a single subsystem.
2. **`wpa_supplicant` (D-Bus-free) + `dhcpcd` + a trusted bridge** — talks to the
   kernel over `nl80211`, exposes a Unix-socket control interface, and a small
   `playos-net` daemon translates that to the existing IPC.
3. **Custom `nl80211` client** — no external dependency, but re-implements years
   of WPA3/SAE roaming and regulatory work. Rejected as unjustifiable risk.

## Decision

Adopt **Option B**: `wpa_supplicant` built **without** its D-Bus control
interface, `dhcpcd` for DHCP, and a new trusted **`playos-net`** bridge daemon
that translates between `wpa_supplicant`'s control protocol (`wpa_ctrl`) /
`dhcpcd` and the `playos-runtime` JSON messages on `control.sock`.

Decomposed as **Sprint 16 — `playos-net`**.

## Rationale

- **Zero D-Bus, zero BusyBox.** `wlroots`/`libseat` already avoid D-Bus, and the
  production image has no BusyBox; this stack keeps both properties.
- **One control plane.** The shell talks only to `control.sock`, exactly like
  `LaunchGame`; `wpa_supplicant` is never exposed to the shell (or games).
- **Smallest trusted surface.** `wpa_supplicant` and `dhcpcd` run as supervised
  children of PID 1; their control sockets live under `/run/playos/net/`, owned
  `root:playos-trusted` `0660`. Games are not in `playos-trusted`, so the path is
  denied by group membership, not by convention.
- **Buildroot support is first-class.** Buildroot ships `wpa_supplicant` with
  selectable `NL80211`, `CTRL_IFACE` (unix socket), `WPA3` (SAE) and
  `WPA_CLIENT_SO` options; D-Bus support is opt-in, so "D-Bus-free" is the
  default we simply do not override. `dhcpcd` and the MediaTek `mt7921`/`mt7922`
  firmware are likewise packaged.

## Limitations Accepted

- **No EAP/enterprise (802.1X)** authentication in this sprint.
- **No Bluetooth.** BlueZ is D-Bus-only; a later, separately-scoped private bus
  is the plan (`network-options.md` §8).
- **No game network access.** Networking is a system/shell capability; a
  per-game allowlist is a later decision.
- **No captive-portal detection, hotspot, Wi-Fi Direct, or mesh.**
- **`wpa_supplicant` is a large-ish C codebase** compared with a custom client;
  accepted as the cost of not re-implementing WPA3/SAE.

## Migration Path

If a future feature genuinely requires D-Bus (Bluetooth being the likely first
case), introduce a **private `dbus-broker` scoped to the trusted zone** rather
than a system bus, and only for the subsystem that needs it. Wi-Fi keeps its
Unix-socket path either way; the public control surface (`ScanNetworks`,
`ConnectNetwork`, `NetworkStatus`, …) does not change, so the shell and any
future consumer are unaffected.

## Consequences

- The Ally kernel config must enable `CFG80211`, `MAC80211`, `MT7921E` and
  `RFKILL` (currently `# CONFIG_WIRELESS is not set`).
- `playos-init` gains supervision of three daemons (`wpa_supplicant`, `dhcpcd`,
  `playos-net`) after `/data` mounts.
- The runtime IPC gains an additive network message set (kept at `"v": 1`);
  documented in `runtime-ipc.md`.
- Network profiles persist under `/data/config/network/`; PSKs must never be
  logged.
- The real scan/connect/DHCP/WPA3 acceptance is **hardware-gated** on the Ally's
  MT7921e; QEMU can only prove the kernel build, daemon startup, and graceful
  scan failure.

## References

- [`../sprints/network-options.md`](../sprints/network-options.md) — full options analysis (§10 recommendation)
- [`../sprints/Sprint-16.md`](../sprints/Sprint-16.md) — the work package
- [`../runtime-ipc.md`](../runtime-ipc.md) — control socket protocol
- [`../kernel-config.md`](../kernel-config.md) — §Networking
