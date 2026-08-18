# Sprint 11 — Immutable Images and A/B Updates

**Goal:** Deliver signed, atomic A/B system updates with automatic rollback on boot failure, on top of the immutable (read-only squashfs) system image installed in Sprint 10.

**Primary Outcome:** The running system image is read-only (games cannot modify it). A system update can be applied to the inactive slot, and after a marked reboot, the device boots from the new slot. If the new slot fails to boot successfully, it rolls back to the previous slot automatically.

**Status:** 🟡 Not started — Sprint 10 (installer + NVMe deploy) is complete; A/B update, boot counting/rollback, and update-signing work has not begun.

**Prerequisites:** Sprint 10 complete — installer creates the disk layout; device boots from internal NVMe.

---

## Why This Sprint Exists

Sprint 10 delivers an installable system with a read-only squashfs root. Sprint 11 adds A/B slot updates so a new system image can be tested in the inactive slot and automatically rolled back if it fails, plus dm-verity integrity hardening. This is the safety foundation for production distribution.

---

## Start Condition Checklist

- Sprint 10 complete: installer creates the full 5-partition layout (ESP, system A/B, `misc`, data); Ally boots from internal NVMe via the ESP EFI artifact.
- System A holds a read-only squashfs root; system B is reserved empty; the `misc` partition exists.
- The EFI boot path (BOOTX64.EFI) is stable.
- An update key pair is generated for development use.
- ADR-0005 (RAUC for A/B System Updates) is Accepted; the RAUC-vs-custom criteria are recorded.

---

## Decisions Locked for This Sprint

- **Read-only mount strategy this sprint:** system slots are inherently read-only squashfs images (no `MS_RDONLY` flag needed); dm-verity is a post-MVP hardening step (document as required for production)
- **A/B slot tracking:** `boot.json` on the ESP (FAT32 writable from `playos-init`); the `misc` partition is reserved as the more robust future home
- **Boot success definition:** shell renders AND user interacts (A/B/D-pad), OR 60-second timer elapses
- **Rollback trigger:** `boot_count >= 3` with `health != "good"` → mark slot bad, switch, reboot
- **Update bundle:** RAUC (or a minimal custom updater, if RAUC integration exceeds the ADR-0005 criteria); development key; production HSM key is post-MVP — final RAUC-vs-custom choice is resolved during S11-T5, consistent with ADR-0005's "pending evaluation" status
- **Network download:** out of scope this sprint; update bundles are placed manually or via USB
- **Installer update:** Sprint 11 does not repartition — Sprint 10 already creates the 5-partition layout; Sprint 11 adds A/B update/rollback logic on top

### Update contract (engine-agnostic, locked)

Fixed regardless of whether RAUC or a custom updater wins in S11-T5 — the shell, `playos-init`, and the engine all build against this contract:

- **Bundle location:** `/data/updates/` directory; bundles are matched by the neutral `.playosb` suffix (never a RAUC-specific extension). Exactly one bundle may be "ready to apply" at a time.
- **`boot.json` schema** — `/EFI/playos/boot.json` on the ESP:
  ```json
  {
    "v": 1,
    "active_slot": "a",
    "slot_a": { "version": "0.1.0", "boot_count": 0, "health": "good" },
    "slot_b": { "version": "", "boot_count": 0, "health": "empty" }
  }
  ```
  `health` ∈ { `good`, `pending`, `bad`, `empty` }. Slot selection, boot counting, and rollback read/write only this file.
- **Apply-update IPC** (shell/overlay → `playos-init`):
  ```json
  { "v": 1, "type": "ApplyUpdate", "path": "/data/updates/0.2.0.playosb" }
  ```
  Responses: `ApplyUpdateAck { "accepted": true }` or `ApplyUpdateError { "reason": "..." }`.
- **Update progress/status events** (`playos-init` → shell/overlay):
  ```json
  { "v": 1, "type": "UpdateProgress", "step": "verify", "percent": 25 }
  { "v": 1, "type": "UpdateComplete", "active_slot": "b", "version": "0.2.0" }
  { "v": 1, "type": "UpdateError", "step": "verify", "reason": "signature_invalid" }
  ```
- **Boot success signal:** the shell reports a successful boot (renders AND user interacts) or a 60-second timer elapses; either resets the active slot's `boot_count` to 0 and sets `health = "good"`.

---

## Scope

### In Scope

- Read-only system partition mount
- A/B update/rollback logic on the existing 5-partition layout (Sprint 10 installer already created the partitions)
- `boot.json` on ESP: active slot, health, boot count
- Boot counting and automatic rollback
- Update flow: signature verify → write inactive slot → update `boot.json` → reboot
- RAUC integration (or equivalent — per ADR)
- Shell update UI (manual trigger; no network download)
- `playos_system_os_version()` API returning active slot version

### Explicitly Out of Scope

- dm-verity (post-MVP)
- Network update download (post-MVP)
- Production HSM update key (post-MVP)
- Delta updates (post-MVP)
- Rollback from within the OS UI (the automatic 3-strike rollback is sufficient this sprint)

---

## Required Repository Changes

| Repo | Required work |
|---|---|
| `playos-refdistro` | RAUC integration, update bundle build target, read-only root image build (A/B layout already from Sprint 10) |
| `playos-init` | Read-only system mount, `boot.json` management, boot counting and rollback, update application flow |
| `playos-shell` | Update UI in settings screen |
| `playos-platform-api` | `playos_system_os_version()` returns active slot version |
| `playos-spec` | A/B boot protocol spec, update flow ADR (RAUC vs custom) |

---

## Expected Files and Directories

### `playos-init`

```text
src/boot_slot.c                      # boot.json reader/writer, slot selection, boot counting
```

### `playos-refdistro`

```text
br2-external/configs/playos_ally_defconfig   # updated: A/B read-only root image
br2-external/configs/playos_ally_installer_defconfig   # updated: 5-partition installer
br2-external/package/rauc/           # RAUC Buildroot package (or custom updater)
scripts/create-update-bundle.sh      # development key signing
```

### `playos-spec`

```text
adr/ADR-0005-update-engine.md        # exists — RAUC vs custom decision; supersede if revised
```

---

## Agent Task Breakdown

### Task Status Grid

| Task ID | Task | Primary repo | Status | Notes / evidence |
|---|---|---|---|---|
| S11-T1 | Implement read-only system partition mount | `playos-init` | not started | |
| S11-T2 | Mount active system slot image and select active slot (layout already created in Sprint 10) | `playos-init` | not started | |
| S11-T3 | Implement `boot.json` read/write and active slot selection | `playos-init` | not started | |
| S11-T4 | Implement boot counting and automatic rollback | `playos-init` | not started | |
| S11-T5 | Integrate RAUC (or equivalent) and update application flow | `playos-refdistro`, `playos-init` | not started | |
| S11-T6 | Implement update bundle signature verification | `playos-init` | not started | |
| S11-T7 | Add shell update UI | `playos-shell` | not started | |
| S11-T8 | Update `playos_system_os_version()` to read active slot version | `playos-platform-api` | not started | API already exists (reads `/etc/playos-version`); must read slot version from `boot.json` |
| S11-T9 | A/B update and rollback validation | `playos-refdistro` | not started | |

### S11-T1 — Mount the read-only system image

- In `playos-init`, mount the active slot's squashfs image read-only:
  ```c
  mount(system_dev, "/", "squashfs", MS_RDONLY, NULL);   /* EROFS is a future hardening option, not implemented this sprint */
  ```
- The image filesystem is inherently read-only; there is no mutable system partition
- All writes at runtime must go to `/data`
- Verify: `touch /usr/test` returns `EROFS` or permission denied
- Log the mount mode on boot: `playos-init: system image mounted read-only`
- Document dm-verity as the production hardening path (do not implement this sprint)

**Done when:** `touch /usr/test` fails with EROFS on a running Ally; all normal operations continue to work.

### S11-T2 — Mount active system slot image and select active slot

The 5-partition layout is created by the Sprint 10 installer and reused unchanged:

```text
GPT disk
├── Part 1: ESP           FAT32            512 MiB    label: ESP
├── Part 2: system A      squashfs   4 GiB      label: playos-a
├── Part 3: system B      squashfs   4 GiB      label: playos-b
├── Part 4: misc          ext4/raw         64 MiB     label: misc
└── Part 5: data          ext4             remainder  label: playos-data
```

- `playos-init` reads `boot.json` (or `misc` metadata) → selects active slot → mounts the matching system image read-only
- Slot B starts as `"health": "empty"` after fresh install
- No repartitioning here — this task consumes the layout Sprint 10 already created
- `misc` is reserved as the more robust home for A/B slot metadata; `boot.json` on the ESP (S11-T3) remains the current implementation

**Done when:** QEMU boots slot A from its read-only squashfs image; `fdisk -l` shows the 5-partition layout.

### S11-T3 — Implement `boot.json` read/write and active slot selection

`boot.json` on ESP at `/EFI/playos/boot.json`:

```json
{
  "active_slot": "a",
  "slot_a": { "version": "0.1.0", "boot_count": 0, "health": "good" },
  "slot_b": { "version": "", "boot_count": 0, "health": "empty" }
}
```

- `playos-init` mounts ESP read-write during boot, reads `boot.json`, unmounts read-only after updating
- Active slot determines which partition label to mount as system
- If `boot.json` is missing or corrupt: boot slot A, log a warning, recreate the file

**Done when:** `cat /EFI/playos/boot.json` after boot shows the correct active slot and boot count.

### S11-T4 — Implement boot counting and automatic rollback

On every boot:
1. Mount ESP read-write
2. Read `boot.json`; increment `boot_count` for the active slot
3. Write updated `boot.json`; unmount ESP
4. If `boot_count >= 3` AND `health != "good"`: mark slot `"health": "bad"`, switch `active_slot` to the other slot, reboot immediately

After successful boot (user interacts with shell OR 60-second timer):
1. Mount ESP read-write
2. Set `health = "good"`, `boot_count = 0` for the active slot
3. Write and unmount

**Done when:** corrupting the active slot's initramfs causes 3 failed boot attempts and then an automatic switch to the other slot.

### S11-T5 — Integrate RAUC (or equivalent) and update application flow

Update application sequence:
1. Receive update bundle path (e.g., `/data/updates/playos-0.2.0.raucb`)
2. Verify bundle signature (see S11-T6)
3. Identify inactive slot (opposite of `active_slot` in `boot.json`)
4. Write new system image to inactive slot partition
5. Update ESP: write new EFI artifact for inactive slot
6. Update `boot.json`: switch `active_slot` to new slot, `boot_count = 0`, `health = "pending"`
7. Notify shell: "Update ready — restart to apply"
8. On restart: new slot boots; rollback guard (S11-T4) runs

RAUC handles steps 2–6 if chosen. Document the integration in the ADR.

**Done when:** applying a valid update bundle causes the inactive slot to be written and `boot.json` to reflect the pending slot switch.

### S11-T6 — Implement update bundle signature verification

- Generate a development key pair: `openssl genrsa -out dev-update.key 4096` + self-signed cert
- Embed the public cert in the system image (e.g., `/etc/playos/update-cert.pem`)
- Before writing any slot, verify the bundle signature against the embedded cert
- On signature failure: abort update, log clearly, do NOT write any partition

**Done when:** a bundle signed with the correct key is accepted; a bundle signed with a wrong key or unsigned is rejected before any write occurs.

### S11-T7 — Add shell update UI

In the shell settings screen:
- "System" section → "Software Update"
- Show: current version (from `playos_system_os_version()`), active slot label
- "Check for Update" button → for this sprint: opens a file picker or shows a "Place bundle in `/data/updates/`" instruction
- "Apply Update" button appears when a bundle is present in `/data/updates/`
- Progress bar during bundle application
- "Restart to Apply" button after bundle written successfully
- Show current slot health in a developer info panel

**Done when:** placing a valid bundle in `/data/updates/` causes the Apply button to appear; applying it shows progress and the restart prompt.

### S11-T8 — Update `playos_system_os_version()` to read active slot version

The API already exists: declared in `playos-platform-api/include/playos/playos_system.h` and implemented in `src/playos_system.c`. It currently reads `/etc/playos-version` (fallback `"0.3.0"`), and the shell already calls it (`src/main.c`, `src/screen_settings.c`, `src/screen_home.c`). This task changes its source to the active slot's version, not its signature:

```c
/* Returns a pointer to a static null-terminated version string, e.g. "0.1.0".
   Returns "unknown" if boot.json cannot be read. */
const char *playos_system_os_version(void);
```

- Read the version of the active slot from `boot.json` on the ESP (S11-T3)
- Keep the existing static-buffer lifetime contract; no caller changes required
- Fall back to `"unknown"` (not a hardcoded version) when `boot.json` is unreadable

**Done when:** `playos_system_os_version()` returns the version string from `boot.json`, and the shell settings/home screens display it unchanged.

### S11-T9 — A/B update and rollback validation

Full test matrix on the ROG Ally:

1. Fresh install → verify 5-partition layout → `boot.json` shows slot A good
2. Apply valid update bundle → `boot.json` shows slot B pending → reboot → slot B active → 60s → slot B good
3. Corrupt slot B initramfs → apply bundle with corrupt slot → reboot 3× → rollback to slot A
4. `/data` content (game files, saves) survives update and rollback unchanged
5. Invalid signature bundle → rejected before any partition write
6. `playos_system_os_version()` returns correct version for each slot

**Done when:** all 6 test cases pass with log evidence.

---

## Implementation Guidance

### ESP mount discipline

The ESP must be mounted read-write ONLY for boot.json updates and EFI artifact writes. Mount it read-only or unmounted the rest of the time. This prevents accidental corruption of the boot files.

### boot.json atomicity

Write `boot.json` atomically: write to `boot.json.tmp`, then rename to `boot.json`. FAT32 rename is not guaranteed atomic on all firmware, but it is the best available option without a journaling filesystem on the ESP.

### RAUC vs custom

If RAUC's Buildroot package is not available or too complex to integrate, implement a minimal custom updater in `playos-init`. The ADR must document the choice and the rationale.

---

## Verification and Evidence

| Evidence | How it is produced |
|---|---|
| Read-only mount | `touch /usr/test` → EROFS; `cat /proc/mounts` showing `ro` |
| A/B layout | `fdisk -l` after fresh install |
| boot.json content | `cat /EFI/playos/boot.json` at each test stage |
| Rollback | 3-strike test log showing slot switch |
| Data survival | File hashes before and after update match |
| Signature rejection | Log showing rejected bundle before any write |

---

## Acceptance Criteria

- [ ] Running system partition mounted read-only; `touch /usr/test` fails with EROFS
- [ ] Installer creates 5-partition layout on fresh NVMe (ESP, A/B system, misc, data)
- [ ] `boot.json` on ESP reflects active slot, health, and boot count
- [ ] Update written to inactive slot does not affect running system
- [ ] After reboot, system boots from newly updated slot
- [ ] Successful boot marks new slot `health = "good"` after 60 seconds
- [ ] 3 consecutive failed boots of new slot triggers rollback to previous slot
- [ ] Previous slot boots and is functional after rollback
- [ ] Games and saves on `/data` survive update and rollback unchanged
- [ ] `playos_system_os_version()` returns active slot version correctly
- [ ] Bundle with invalid signature is rejected before any partition write
- [ ] Shell update UI shows current version and allows applying a bundle
- [ ] CI: update bundle creation and signature verification tested on host

---

## Handoff to Sprint 12

Sprint 12 (Security Hardening) may assume:

- The system image is immutable at runtime (read-only system mount)
- Signed A/B updates with automatic rollback are functional
- `boot.json` on the ESP tracks the active slot, slot health, and boot count
- Games and user data live on `/data` and are unaffected by updates and rollback

---

## Exit Gate

The system image is immutable at runtime. A/B updates can be applied, verified, and automatically rolled back on failure. Games and user data are unaffected by updates and rollbacks.

*Previous: [Sprint 10](Sprint-10.md) | Next: [Sprint 12](Sprint-12.md)*
