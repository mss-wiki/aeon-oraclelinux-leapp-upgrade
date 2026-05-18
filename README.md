# Oracle Linux LEAPP Upgrade: OL7 → OL8 Troubleshooting

Documentation for post-LEAPP upgrade boot failures caused by volume group name mismatches in GRUB configuration.

---

## Problem

After completing a LEAPP in-place upgrade from Oracle Linux 7 to Oracle Linux 8, the system fails to boot with a **dracut error** — the volume group is not activated during the initramfs phase.

**Root cause area:** GRUB's kernel command line references a volume group name (`rhel`) that does not match what dracut finds on disk (`ol`), or the VG exists but cannot be activated due to a stale initramfs or LVM filter.

Typical error seen during boot:
```
Warning: /dev/rhel/root does not exist
Warning: /dev/rhel/swap does not exist
Dropping to debug shell.
dracut:/#
```

---

## Why This Happens

LEAPP replaces bootloader configs and kernel packages during the upgrade. Common scenarios that produce this mismatch:

| Scenario | Cause |
|---|---|
| System was originally RHEL, converted to OL | VG named `rhel` from RHEL install; LEAPP updated GRUB to reference `ol` |
| Fresh OL7 install with default VG name `ol` | LEAPP failed to regenerate GRUB/initramfs correctly |
| Interrupted LEAPP run | GRUB was partially updated, leaving inconsistent VG references |

---

## Phase 1 — Get Into the System

You need shell access before making any changes. Use the first method that works.

### Method A: Edit GRUB temporarily (fastest, no media needed)

1. At the GRUB menu, highlight the OL8 kernel entry and press **`e`**
2. Find the `linux` line and locate these parameters:
   ```
   root=/dev/mapper/rhel-root
   rd.lvm.lv=rhel/root
   rd.lvm.lv=rhel/swap
   ```
3. Change `rhel` → `ol` in all occurrences (adjust if your actual VG name differs)
4. Press **Ctrl+X** to boot with the modified parameters
5. If the system boots → proceed to [Permanent Fix](#phase-3--permanent-fix)

### Method B: Dracut emergency shell

If the system drops you to a dracut emergency shell:
```bash
# Activate all VGs to see what's on disk
vgchange -ay

# If the VG activates, exit the shell to continue boot
exit
```

### Method C: Rescue mode from OL8 ISO

1. Boot from OL8 installation media
2. Select **Troubleshooting → Rescue a Red Hat/Oracle Linux system**
3. Choose option 1 (mount under `/mnt/sysimage`)
4. Chroot into the system:
   ```bash
   chroot /mnt/sysimage
   ```

---

## Phase 2 — Gather Evidence

From whichever shell you obtain, collect this information **before making any changes**.

```bash
# 1. What VGs actually exist on disk?
vgs -o name,uuid,attr

# 2. What LVs exist?
lvs -o name,vgname,path

# 3. What does GRUB currently reference?
grep -E "rd.lvm|root=" /boot/grub2/grub.cfg | head -20

# 4. What kernel cmdline was used for this boot?
cat /proc/cmdline

# 5. What does /etc/default/grub say?
cat /etc/default/grub

# 6. Check /etc/fstab for device references
grep -v "^#" /etc/fstab

# 7. Check if dracut has the LVM module
dracut --list-modules 2>/dev/null | grep lvm
```

### Interpreting the Output

| `vgs` shows | GRUB references | Diagnosis |
|---|---|---|
| `ol` | `rhel` | GRUB has wrong VG name → [Fix A](#fix-a-grub-references-wrong-vg-name) |
| `rhel` | `rhel` | GRUB is correct, initramfs is stale → [Fix B](#fix-b-regenerate-initramfs) |
| `rhel` | `ol` | VG needs renaming or GRUB needs reverting → [Fix C](#fix-c-rename-the-vg) |
| VG exists, not activating | — | VG deactivated or corrupt → [Fix D](#fix-d-force-vg-activation) |

---

## Phase 3 — Permanent Fix

Apply only after confirming the diagnosis from Phase 2.

### Fix A: GRUB references wrong VG name

Use when `vgs` shows `ol` but GRUB has `rhel`.

```bash
# Edit GRUB default config — update GRUB_CMDLINE_LINUX
vi /etc/default/grub
# Change: rd.lvm.lv=rhel/root  →  rd.lvm.lv=ol/root
# Change: rd.lvm.lv=rhel/swap  →  rd.lvm.lv=ol/swap

# Regenerate GRUB config
grub2-mkconfig -o /boot/grub2/grub.cfg

# Regenerate initramfs for the current kernel
dracut --force --verbose /boot/initramfs-$(uname -r).img $(uname -r)

# Reboot to verify
reboot
```

### Fix B: Regenerate initramfs

Use when GRUB is correct but dracut still can't activate the VG (stale initramfs from OL7).

```bash
# Force rebuild initramfs with LVM support
dracut --force --add lvm /boot/initramfs-$(uname -r).img $(uname -r)

# Reboot to verify
reboot
```

### Fix C: Rename the VG

Use when the VG is named `rhel` on disk but GRUB references `ol`. Renaming the VG avoids touching GRUB/fstab.

> **Warning:** After renaming, update `/etc/fstab` — any `/dev/mapper/rhel-*` entries must change to `/dev/mapper/ol-*`.

```bash
# Rename VG
vgrename rhel ol

# Update /etc/fstab
sed -i 's|/dev/mapper/rhel-|/dev/mapper/ol-|g' /etc/fstab

# Regenerate GRUB and initramfs
grub2-mkconfig -o /boot/grub2/grub.cfg
dracut --force /boot/initramfs-$(uname -r).img $(uname -r)

# Reboot to verify
reboot
```

### Fix D: Force VG activation

Use when the VG exists but is in an inactive/partial state.

```bash
# Force-activate all VGs
vgchange -ay

# Check for partial VG (missing PVs)
vgs -o name,attr,missing_pv

# If VG is partial and you're sure PVs are present
vgchange --partial --activate y <vgname>

# If VG activates successfully, rebuild initramfs
dracut --force /boot/initramfs-$(uname -r).img $(uname -r)
reboot
```

---

## Post-Fix Verification

After a successful reboot, verify the system is healthy:

```bash
# Confirm running OL8
cat /etc/oracle-release
uname -r

# Confirm VG is active and LVs are mounted
vgs
lvs
df -h

# Check for any remaining LEAPP leftovers
rpm -qa | grep leapp
ls /var/log/leapp/

# Review LEAPP upgrade report
cat /var/log/leapp/leapp-report.txt | grep -E "WARN|ERROR"
```

---

## References

- [Oracle LEAPP Upgrade Documentation](https://docs.oracle.com/en/operating-systems/oracle-linux/8/leapp/)
- [Red Hat KB: dracut emergency shell at boot](https://access.redhat.com/solutions/1588483)
- `man vgrename` — LVM volume group rename
- `man dracut` — initramfs generation
