# PlayOS Installer Specification

The installer writes PlayOS to a fixed disk. It is the only component that
destroys data, so it is deliberately small, linear and explicit.

## Components

| Binary | UI | Used for |
|---|---|---|
| `playos-installer` | Raylib/Wayland window | `playos.mode=install`, `playos.install.auto=1`, recovery, and any install where no shell is running to draw progress |
| `playos-install-worker` | none | installs driven from the shell (Sprint 14.5) |
| `libplayos-install` | none | the step sequence both binaries run — one engine, no forked logic |

`libplayos-install` (Sprint 14.5-T1) owns the eight steps and their order:

| # | Step | Does |
|---|---|---|
| 0 | Create GPT | releases the target's mounts, writes the partition table |
| 1 | Format ESP | `mkfs.fat`, label `ESP` |
| 2 | Write system A | writes `rootfs.squashfs` to `playos-a` |
| 3 | Reserve system B | no-op; `playos-b` is reserved for the A/B updater |
| 4 | Format misc | `ext4`, label `misc` |
| 5 | Format data | `ext4`, label `playos-data`, seeds the dev SSH key when one is available |
| 6 | Write EFI | copies the boot kernel and `/EFI/playos/boot.json` to the target's ESP |
| 7 | Sync | flushes |

The engine has no UI, no logging sink and no device discovery of its own: callers
pass a context plus optional log, sub-step progress and seed-key callbacks. Step
names and per-step log lines are stable, because logs, docs and tests compare
them.

## Screen-less worker (Sprint 14.5-T3)

When the shell drives an install it does not hand the session over. init spawns
`playos-install-worker` and the shell keeps the screen:

- **Screen-less by construction** — the worker links only `libfdisk` and libc (no
  Wayland, raylib or EGL), so it cannot create a surface even by accident; its
  `NEEDED` list is checked when it is built.
- **Nothing is torn down** — the compositor, `/data`, the shell, the overlay and
  SSH all stay up. The standalone handoff remains for the cases where nobody can
  draw progress.
- **init owns the payload** — init mounts the boot medium's `playos-a` partition
  at `/mnt/payload` and passes `PLAYOS_INSTALL_PAYLOAD`, so the worker needs no
  discovery; that stays in the standalone installer.
- **Progress is relayed, not dialled** — the worker sends `InstallProgress`,
  `InstallComplete` and `InstallError` to init, which forwards them to the
  registered shell listener (`runtime-ipc.md`).
- **A dead worker is an error, not a stall** — on a non-zero exit or a signal,
  init emits `InstallError`, because the shell has no other way to learn the
  worker is gone.
- **No reboot** — the worker exits cleanly and the shell offers "Reboot now".

## Partition layout it writes

Five partitions on the target (see `partition-layout.md`): `ESP` (512 MiB,
FAT32), `playos-a` and `playos-b` (4 GiB each, immutable squashfs), `misc`
(64 MiB, ext4, A/B metadata) and `playos-data` (remainder, ext4). The layout is
fixed: the A/B updater depends on it, and changing it requires an ADR.

## Requirements

- Runs as root; drives `mkfs.fat`, `mkfs.ext4`, `libfdisk` and `blockdev`.
- Refuses to install to the disk holding the running root or `/data`
  (`PrepareInstall` in init rejects it, and the picker hides it).
- The target's mounts must be released first: a mounted ESP makes `mkfs` refuse
  the target and the kernel refuse to re-read the partition table.
