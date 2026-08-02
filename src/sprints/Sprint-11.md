# Sprint 11 — Immutable Images and A/B Updates

**Goal:** Make the PlayOS system image immutable (read-only), implement an A/B partition scheme, and deliver signed, atomic system updates with automatic rollback on boot failure.

**Primary Outcome:** The running system image is read-only (games cannot modify it). A system update can be applied to the inactive slot, and after a marked reboot, the device boots from the new slot. If the new slot fails to boot successfully, it rolls back to the previous slot automatically.

**Prerequisites:** Sprint 10 complete — installer creates the disk layout; device boots from internal NVMe.

---

## Why This Sprint Exists

Sprint 10 delivers an installable system but the installed image is mutable — anything running as root can modify the system partition. Sprint 11 makes the system image read-only and adds A/B slots so updates can be tested in the inactive slot and automatically rolled back if they fail. This is the safety foundation for production distribution.

---

## Start Condition Checklist

- Sprint 10 complete: installer creates a 3-partition layout; Ally boots from internal NVMe.
- `playos-init` reads and mounts the system partition.
- The EFI boot path (BOOTX64.EFI) is stable.
- An update key pair is generated for development use.
- RAUC evaluated and chosen or a custom updater decision is documented in an ADR.

---

## Decisions Locked for This Sprint

- **Read-only mount strategy this sprint:** simple `MS_RDONLY` ext4 mount; dm-verity is a post-MVP hardening step (document as required for production)
- **A/B slot tracking:** `boot.json` on the ESP (FAT32 writable from `playos-init`)
- **Boot success definition:** shell renders AND user interacts (A/B/D-pad), OR 60-second timer elapses
- **Rollback trigger:** `boot_count >= 3` with `health != "good"` → mark slot bad, switch, reboot
- **Update bundle:** RAUC format with development key; production HSM key is post-MVP
- **Network download:** out of scope this sprint; update bundles are placed manually or via USB
- **Installer update:** Sprint 11 updates the installer to create 4-partition (A/B) layout on fresh installs

---

## Scope

### In Scope

- Read-only system partition mount
- A/B partition layout (updated installer and `playos-init`)
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
| `playos-refdistro` | A/B partition layout in installer, `playos-init` boot.json management, RAUC integration, update bundle build target |
| `playos-shell` | Update UI in settings screen |
| `playos-platform-api` | `playos_system_os_version()` returns active slot version |
| `playos-spec` | A/B boot protocol spec, update flow ADR (RAUC vs custom) |

---

## Expected Files and Directories

### `playos-refdistro`

```text
src/playos-init/src/boot_slot.c      # boot.json reader/writer, slot selection, boot counting
br2-external/configs/playos_ally_defconfig   # updated: 4-partition layout
br2-external/configs/playos_ally_installer_defconfig   # updated: 4-partition installer
br2-external/package/rauc/           # RAUC Buildroot package (or custom updater)
scripts/create-update-bundle.sh      # development key signing
```

### `playos-spec`

```text
adr/ADR-0006-update-mechanism.md     # RAUC vs custom decision
```

---

## Agent Task Breakdown

### Task Status Grid

| Task ID | Task | Primary repo | Status | Notes / evidence |
|---|---|---|---|---|
| S11-T1 | Implement read-only system partition mount | `playos-refdistro` | not started | |
| S11-T2 | Update installer and `playos-init` for A/B partition layout | `playos-refdistro` | not started | |
| S11-T3 | Implement `boot.json` read/write and active slot selection | `playos-refdistro` | not started | |
| S11-T4 | Implement boot counting and automatic rollback | `playos-refdistro` | not started | |
| S11-T5 | Integrate RAUC (or equivalent) and update application flow | `playos-refdistro` | not started | |
| S11-T6 | Implement update bundle signature verification | `playos-refdistro` | not started | |
| S11-T7 | Add shell update UI | `playos-shell` | not started | |
| S11-T8 | Add `playos_system_os_version()` API | `playos-platform-api` | not started | |
| S11-T9 | A/B update and rollback validation | `playos-refdistro` | not started | |

### S11-T1 — Implement read-only system partition mount

- In `playos-init`, mount the system partition with `MS_RDONLY`:
  ```c
  mount(system_dev, "/", "ext4", MS_RDONLY, NULL);
  ```
- All writes at runtime must go to `/data`
- Verify: `touch /usr/test` returns `EROFS` or permission denied
- Log the mount mode on boot: `playos-init: system partition mounted read-only`
- Document dm-verity as the production hardening path (do not implement this sprint)

**Done when:** `touch /usr/test` fails with EROFS on a running Ally; all normal operations continue to work.

### S11-T2 — Update installer and `playos-init` for A/B partition layout

New 4-partition layout:

```
GPT disk
├── Part 1: EFI System Partition    FAT32    512 MB    label: EFI
├── Part 2: PlayOS system A         ext4     2 GB      label: playos-system-a
├── Part 3: PlayOS system B         ext4     2 GB      label: playos-system-b
└── Part 4: PlayOS data             ext4     remainder  label: playos-data
```

- Update `playos-installer` to create 4 partitions
- Update `playos-init` to: read `boot.json` → select active slot → mount the correct partition
- Slot B starts as `"health": "empty"` after fresh install
- Update both `playos_ally_defconfig` and `playos_ally_installer_defconfig`

**Done when:** fresh install creates 4-partition layout; QEMU boots slot A; `fdisk -l` shows correct layout.

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

### S11-T8 — Add `playos_system_os_version()` API

```c
/* Returns a pointer to a static null-terminated version string, e.g. "0.1.0".
   Returns "unknown" if boot.json cannot be read. */
const char *playos_system_os_version(void);
```

- Reads the version of the active slot from `boot.json` on the ESP
- Called by the shell settings screen and overlay system info section

**Done when:** `playos_system_os_version()` returns the correct version string that matches `boot.json`.

### S11-T9 — A/B update and rollback validation

Full test matrix on the ROG Ally:

1. Fresh install → verify 4-partition layout → `boot.json` shows slot A good
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
- [ ] Installer creates 4-partition A/B layout on fresh NVMe
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

## Handoff (MVP Complete)

Sprint 11 is the final MVP sprint. After this sprint:

- The system is immutable, installable, and self-updating
- A/B updates with automatic rollback are functional
- All sprints 0–11 are complete
- Post-MVP work begins: network update download, dm-verity, Bluetooth, cloud saves, marketplace

---

## Exit Gate

The system image is immutable at runtime. A/B updates can be applied, verified, and automatically rolled back on failure. Games and user data are unaffected by updates and rollbacks.

*Previous: [Sprint 10](Sprint-10.md) | Next: [Sprint 12](Sprint-12.md)*

**Make the system partition read-only:**

Option A: Mount with `ro` flag in `playos-init`
```c
mount(system_dev, "/", "ext4", MS_RDONLY, NULL);
```

Option B: Use dm-verity (preferred for production — detects tampering):
- Append a verity hash tree to the system image at build time (`veritysetup format`)
- `playos-init` sets up the dm-verity device and mounts it read-only
- Any modification attempt returns `EROFS`
- dm-verity failures cause `playos-init` to refuse to boot that slot

For this sprint: implement Option A (simple read-only mount). Document Option B (dm-verity) as a Sprint 12/production requirement.

**Consequences:**
- Kernel, initramfs, compositor, shell, `libplayos`, firmware, and core assets are read-only at runtime
- All runtime writes go to `/data` (already enforced by architecture)
- Verify: `touch /usr/test` returns `EROFS` or permission denied
- Games, saves, logs, cache, config — all on `/data` — are unaffected

### A/B Partition Layout

Expand the Sprint 10 layout to A/B:

```
GPT disk
├── Part 1: EFI System Partition    FAT32    512 MB    label: EFI
├── Part 2: PlayOS system A         ext4     2 GB      label: playos-system-a
├── Part 3: PlayOS system B         ext4     2 GB      label: playos-system-b
└── Part 4: PlayOS data             ext4     remainder  label: playos-data
```

**ESP contains two boot entries:**
```
/EFI/BOOT/BOOTX64.EFI         → delegates to active slot
/EFI/playos/slotA.efi         → slot A kernel + initramfs
/EFI/playos/slotB.efi         → slot B kernel + initramfs
```

Or: Use a single EFI artifact with a slot variable stored in UEFI NVRAM or a flag file on the ESP.

**Active slot tracking:**
Store in `/EFI/playos/boot.json` on the ESP (writable from `playos-init` for boot counting):

```json
{
  "active_slot": "a",
  "slot_a": { "version": "0.1.0", "boot_count": 0, "health": "good" },
  "slot_b": { "version": "", "boot_count": 0, "health": "empty" }
}
```

`playos-init` reads `boot.json` at boot to determine which slot to mount.

### Boot Counting and Automatic Rollback

**On every boot:**
1. `playos-init` reads `boot.json`
2. Increments `boot_count` for the active slot
3. If `boot_count >= 3` and `health != "good"`: mark slot as `"health": "bad"`, switch active slot to the other one, reboot
4. After a successful boot (shell rendered and user interacted OR 60-second timer): `playos-init` marks `health = "good"` and resets `boot_count = 0`

This provides automatic rollback after up to 3 failed boot attempts.

### Update Flow

**Update engine:** Use RAUC unless a simpler EFI-image-specific updater is sufficient. Document the choice as an ADR.

**Full update flow:**
1. Download signed update bundle to `/data/updates/`
2. Verify signature (public key embedded in current system image)
3. Identify the inactive slot (opposite of `active_slot` in `boot.json`)
4. Write new system image to inactive slot's partition
5. Update `boot.json`: set `active_slot` to the new slot, `boot_count = 0`, `health = "pending"`
6. Display "Update ready — restart to apply" in the shell/overlay
7. User triggers restart (or automatic)
8. System boots from new slot
9. If successful: `health = "good"` after 60 seconds
10. If boot fails 3 times: rollback to previous slot

**Update bundle format (RAUC):**
- Signed with a PlayOS update key
- Contains: kernel+initramfs EFI artifact + system partition image OR delta
- Bundle metadata: version, min supported current version, changelog

**Update key management:**
- Development builds: use a development key (not secret, used for testing)
- Production builds: use a production HSM-backed key (post-MVP)
- The verification public key is embedded in `libplayos` or the initramfs

### Shell and Overlay Update UI

**Shell settings screen → "System Update":**
- Current version, last check time
- "Check for Update" button (stub — manual trigger for this sprint)
- Download and apply progress
- "Restart to Apply" button after download

**For this sprint:** Trigger updates manually via a USB-placed update bundle or a local file. Network update download is post-MVP.

---

## Acceptance Criteria

- [ ] Running system partition is mounted read-only; `touch /usr/test` fails with `EROFS`
- [ ] A/B partition layout created by updated installer
- [ ] `boot.json` on ESP reflects active slot and health state
- [ ] Update applied to inactive slot does not affect the running system
- [ ] After reboot, system boots from the newly updated slot
- [ ] New slot boots successfully → `health = "good"` written after 60 seconds
- [ ] Simulated boot failure (corrupt initramfs in new slot): boot count reaches 3, rolls back to old slot
- [ ] Rollback boots the old slot; system is functional
- [ ] Games and saves on `/data` are untouched by the update and rollback
- [ ] System version reported by `playos_system_os_version()` reflects the active slot version
- [ ] Installer creates A/B layout on fresh NVMe install
- [ ] CI: update bundle creation and signature verification tested on host

---

## Repositories Primarily Involved

| Repo | Work |
|---|---|
| `playos-refdistro` | A/B partition layout, boot.json management in `playos-init`, RAUC integration, update bundle build target |
| `playos-shell` | Update UI in settings screen |
| `playos-platform-api` | `playos_system_os_version()` returns active slot version |
| `playos-spec` | Update flow ADR (RAUC vs custom), A/B boot protocol spec |

---

## Testing Approach

- Physical ROG Ally: full A/B update cycle
- Rollback test: corrupt the new slot's initramfs; verify 3-strike rollback to old slot
- Data persistence test: create files in `/data` before update; verify they survive
- QEMU: A/B boot logic, boot counting, rollback on loopback disks
- Signature test: verify a bundle with an invalid signature is rejected before any write

---

## Exit Gate

The system image is immutable at runtime. A/B updates can be applied, verified, and automatically rolled back on failure. Games and user data are unaffected by updates and rollbacks.

*Previous: [Sprint 10](Sprint-10.md) | Next: [Sprint 12](Sprint-12.md)*
