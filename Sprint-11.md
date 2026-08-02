# Sprint 11 — Immutable Images and A/B Updates

**Goal:** Make the PlayOS system image immutable (read-only), implement an A/B partition scheme, and deliver signed, atomic system updates with automatic rollback on boot failure.

**Primary Outcome:** The running system image is read-only (games cannot modify it). A system update can be applied to the inactive slot, and after a marked reboot, the device boots from the new slot. If the new slot fails to boot successfully, it rolls back to the previous slot automatically.

**Prerequisites:** Sprint 10 complete — installer creates the disk layout; device boots from internal NVMe.

---

## Key Deliverables

### Immutable System Image

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
