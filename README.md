# Oracle Linux In-Place Upgrade: OL7 → OL8 with LEAPP

Complete guide covering the LEAPP upgrade procedure from Oracle Linux 7 to OL8, and troubleshooting post-upgrade boot failures caused by volume group name mismatches.

---

## Table of Contents

1. [Prerequisites](#prerequisites)
2. [Upgrade Steps](#upgrade-steps)
3. [Post-Upgrade Tasks](#post-upgrade-tasks)
4. [Troubleshooting: Boot Failure (dracut / VG not active)](#troubleshooting-boot-failure)
5. [References](#references)

---

## Prerequisites

Before starting the upgrade, verify all of the following:

### System Requirements

| Requirement | Details |
|---|---|
| Source OS version | Oracle Linux **7.9** (latest point release — LEAPP requires 7.9) |
| Architecture | x86_64 only |
| Free disk space | Minimum 10 GB free on `/` and `/var` |
| Secure Boot | Must be **disabled** |
| Kernel | Must be booted into the latest OL7 kernel |
| Network | Access to Oracle public yum or ULN repositories |

### Pre-Upgrade Checklist

- [ ] Take a full snapshot or backup of the system before proceeding
- [ ] Confirm you are on OL7.9: `cat /etc/oracle-release`
- [ ] Ensure no in-progress `yum` transactions: `yum history`
- [ ] Disable third-party repositories (re-enable after upgrade if needed)
- [ ] Note any custom kernel modules — these will need rebuilding after upgrade
- [ ] Verify sufficient disk space: `df -h /`

---

## Upgrade Steps

### Step 1: Update the System to Latest OL7 Packages

```bash
yum update -y
reboot
```

After reboot, confirm the latest kernel is running:

```bash
uname -r
rpm -q kernel | tail -1
```

### Step 2: Enable the LEAPP Repository

**Oracle public yum:**
```bash
yum-config-manager --enable ol7_leapp
# Also enable ol7_addons if not already enabled
yum-config-manager --enable ol7_addons
```

**ULN (Unbreakable Linux Network):**
```bash
# Subscribe the system to the ol7_leapp channel via ULN web interface
# or using uln-channel:
uln-channel --add --channel ol7_x86_64_leapp
```

### Step 3: Install LEAPP and Upgrade Data

```bash
yum install -y leapp leapp-upgrade-el7toel8
```

Verify the required data files are present:

```bash
ls /etc/leapp/files/
# Expected: leapp_upgrade_repositories.repo  pes-events.json  product_type
```

If the data files are missing, download them from Oracle:

```bash
# Download LEAPP data package
yum install -y leapp-data-oraclelinux
```

### Step 4: Run the Pre-Upgrade Assessment

This generates a report of blockers and warnings without making any changes:

```bash
leapp preupgrade --target 8.10
```

Review the report:

```bash
cat /var/log/leapp/leapp-report.txt
```

The report uses three severity levels:

| Level | Meaning |
|---|---|
| **inhibitor** | Blocks the upgrade — must be resolved before proceeding |
| **high** | Serious issue, strongly recommended to fix |
| **medium/low** | Warnings that can be addressed after upgrade |

### Step 5: Resolve Inhibitors

Common inhibitors and their fixes:

**Unsupported or removed packages**
```bash
# List packages LEAPP flagged as unsupported
grep -A5 "inhibitor" /var/log/leapp/leapp-report.txt

# Remove flagged packages (example)
yum remove -y <package-name>
```

**BIOS boot — missing `/boot/efi` partition**
```bash
# For BIOS (non-EFI) systems, LEAPP may need this flag
# Add to leapp answer file:
leapp answer --section remove_pam_pkcs11_module_check.confirm=True
```

**Third-party or unsigned kernel modules**
```bash
# List loaded non-Oracle modules
lsmod | grep -v "^$(uname -r)"

# Unload or blacklist before upgrade if needed
modprobe -r <module_name>
echo "blacklist <module_name>" >> /etc/modprobe.d/blacklist.conf
```

**Network configuration issues**
```bash
# LEAPP requires NM-controlled interfaces; fix legacy ifcfg files
ls /etc/sysconfig/network-scripts/ifcfg-*
# Ensure each has: NM_CONTROLLED=yes
```

**VDO devices present**
```bash
# If VDO is in use, LEAPP may block — check the report for guidance
```

Re-run preupgrade after resolving inhibitors until the report has no inhibitors:

```bash
leapp preupgrade --target 8.10
```

### Step 6: Run the Upgrade

Once the pre-upgrade report is clean, start the upgrade:

```bash
leapp upgrade --target 8.10
```

This command:
1. Downloads all required OL8 packages
2. Prepares a special initramfs for the upgrade transaction
3. Prompts you to reboot when ready

Expected output on success:
```
==> Processing phase `InterimPreparation`
...
==> Processing phase `DownloadBaseOs`
...
A reboot is required to continue. Please reboot your system.
```

### Step 7: Reboot to Complete the Upgrade

```bash
reboot
```

**What happens during reboot (do not interrupt):**

1. System boots into the LEAPP upgrade initramfs
2. The full RPM upgrade transaction runs (this can take 15–45 minutes)
3. System configuration is migrated (bootloader, network, SELinux)
4. System reboots automatically into OL8

You can monitor progress on the console. The system will reboot twice — once for the LEAPP transaction, once into the final OL8 environment.

---

## Post-Upgrade Tasks

Once the system is booted into OL8:

### Verify the Upgrade

```bash
# Confirm OL8
cat /etc/oracle-release
uname -r

# Check all services are running
systemctl --failed

# Review the LEAPP upgrade report for post-upgrade notes
cat /var/log/leapp/leapp-report.txt | grep -E "WARN|ERROR|inhibitor"
```

### Clean Up LEAPP Packages

```bash
yum remove -y leapp leapp-upgrade-el7toel8 leapp-data-oraclelinux

# Remove old OL7 kernels
rpm -q kernel
yum remove -y kernel-<old-ol7-version>
```

### Update All Packages to Latest OL8

```bash
yum update -y
```

### Re-enable Third-Party Repos

If you disabled any repos before the upgrade, re-enable them now — but verify they have OL8-compatible packages first.

### Rebuild Custom Kernel Modules

If you had custom or DKMS modules, rebuild them against the OL8 kernel:

```bash
# For DKMS modules
dkms autoinstall
```

---

## Troubleshooting: Boot Failure

### Problem

After LEAPP upgrade, the system fails to boot with a **dracut error** — the volume group is not activated during the initramfs phase.

**Root cause area:** GRUB's kernel command line references a volume group name (`rhel`) that does not match what dracut finds on disk (`ol`), or the VG exists but cannot be activated due to a stale initramfs or LVM filter.

Typical error seen during boot:
```
Warning: /dev/rhel/root does not exist
Warning: /dev/rhel/swap does not exist
Dropping to debug shell.
dracut:/#
```

### Why This Happens

LEAPP replaces bootloader configs and kernel packages during the upgrade. Common scenarios that produce this mismatch:

| Scenario | Cause |
|---|---|
| System was originally RHEL, converted to OL | VG named `rhel` from RHEL install; LEAPP updated GRUB to reference `ol` |
| Fresh OL7 install with default VG name `ol` | LEAPP failed to regenerate GRUB/initramfs correctly |
| Interrupted LEAPP run | GRUB was partially updated, leaving inconsistent VG references |

### Phase 1 — Get Into the System

You need shell access before making any changes. Use the first method that works.

**Method A: Edit GRUB temporarily (fastest, no media needed)**

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

**Method B: Dracut emergency shell**

If the system drops you to a dracut emergency shell:
```bash
# Activate all VGs to see what's on disk
vgchange -ay

# If the VG activates, exit the shell to continue boot
exit
```

**Method C: Rescue mode from OL8 ISO**

1. Boot from OL8 installation media
2. Select **Troubleshooting → Rescue a Red Hat/Oracle Linux system**
3. Choose option 1 (mount under `/mnt/sysimage`)
4. Chroot into the system:
   ```bash
   chroot /mnt/sysimage
   ```

### Phase 2 — Gather Evidence

From whichever shell you obtain, collect this information **before making any changes**:

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

**Interpreting the output:**

| `vgs` shows | GRUB references | Diagnosis |
|---|---|---|
| `ol` | `rhel` | GRUB has wrong VG name → Fix A |
| `rhel` | `rhel` | GRUB is correct, initramfs is stale → Fix B |
| `rhel` | `ol` | VG needs renaming or GRUB needs reverting → Fix C |
| VG exists, not activating | — | VG deactivated or corrupt → Fix D |

### Phase 3 — Permanent Fix

Apply only after confirming the diagnosis from Phase 2.

**Fix A: GRUB references wrong VG name**

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

reboot
```

**Fix B: Regenerate initramfs**

Use when GRUB is correct but dracut still can't activate the VG (stale initramfs from OL7).

```bash
dracut --force --add lvm /boot/initramfs-$(uname -r).img $(uname -r)
reboot
```

**Fix C: Rename the VG**

Use when the VG is named `rhel` on disk but GRUB references `ol`.

> **Warning:** After renaming, update `/etc/fstab` — any `/dev/mapper/rhel-*` entries must change to `/dev/mapper/ol-*`.

```bash
vgrename rhel ol
sed -i 's|/dev/mapper/rhel-|/dev/mapper/ol-|g' /etc/fstab
grub2-mkconfig -o /boot/grub2/grub.cfg
dracut --force /boot/initramfs-$(uname -r).img $(uname -r)
reboot
```

**Fix D: Force VG activation**

Use when the VG exists but is in an inactive or partial state.

```bash
# Force-activate all VGs
vgchange -ay

# Check for partial VG (missing PVs)
vgs -o name,attr,missing_pv

# If VG is partial and PVs are physically present
vgchange --partial --activate y <vgname>

# Rebuild initramfs if VG activates successfully
dracut --force /boot/initramfs-$(uname -r).img $(uname -r)
reboot
```

### Post-Fix Verification

```bash
# Confirm running OL8
cat /etc/oracle-release
uname -r

# Confirm VG is active and LVs are mounted
vgs && lvs && df -h

# Check for any remaining LEAPP leftovers
rpm -qa | grep leapp

# Review LEAPP upgrade report
grep -E "WARN|ERROR" /var/log/leapp/leapp-report.txt
```

---

## References

- [Oracle LEAPP Upgrade Documentation](https://docs.oracle.com/en/operating-systems/oracle-linux/8/leapp/)
- [Oracle Linux Blog: Upgrading OL7 to OL8](https://blogs.oracle.com/linux/)
- [Red Hat KB: dracut emergency shell at boot](https://access.redhat.com/solutions/1588483)
- `man leapp` — LEAPP upgrade tool
- `man vgrename` — LVM volume group rename
- `man dracut` — initramfs generation
