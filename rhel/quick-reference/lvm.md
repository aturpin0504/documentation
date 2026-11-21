---
layout: default
title: LVM
parent: Quick Reference
grand_parent: RHEL
nav_order: 16
---

# Understanding Logical Volume Manager (LVM)

## LVM Layers

| Layer | Description |
|---|---|
| **Physical Volume (PV)** | A disk or partition initialized for LVM use. |
| **Volume Group (VG)** | A pool of storage created from one or more physical volumes. |
| **Logical Volume (LV)** | A virtual partition created from available space in a volume group. |

LVM is the default storage configuration for many RHEL installations (commonly used for `/` and swap, but not `/boot`) and supports online resizing and adding capacity.

## Viewing Storage

```bash
# List block devices and hierarchy
lsblk

# Show physical volume, volume group, and logical volume summaries
pvs
vgs
lvs

# Display UUIDs for /etc/fstab
blkid

# Show file system disk usage
df -h
```

For LVM-backed entries in `/etc/fstab`, reference either `/dev/vgname/lvname` (from `lvs`) or `UUID="..."` (from `blkid`).

## Creating New LVM Storage

```bash
# 1. View the new disk
lsblk

# 2. Create a volume group
vgcreate vgdata /dev/sdb

# 3. Create a 10 GB logical volume
lvcreate -n lvdata -L 10G vgdata

# 4. Create an XFS file system
mkfs.xfs /dev/vgdata/lvdata

# 5. Create the mount point and add an /etc/fstab entry
mkdir /data
# /dev/vgdata/lvdata /data xfs defaults 0 2

# 6. Mount all /etc/fstab entries and verify
mount -a
df -h /data
```

## VMware: Extending an Existing VMDK

When a VMware virtual disk (`/dev/sda`) is expanded in vSphere while the VM remains online:

```bash
# 1. Rescan the disk for the new size and verify
echo 1 > /sys/class/block/sda/device/rescan
lsblk

# 2. Resize the physical volume and confirm free VG space
pvresize /dev/sda2
vgs

# 3. Extend the logical volume and file system, then verify
lvextend -l +100%FREE -r /dev/vgdata/lvdata
df -h /data
```

## VMware: Adding a New Disk to an Existing Volume Group

When a new disk (`/dev/sdc`) is attached to a running VM:

```bash
# 1. Rescan the SCSI bus (if needed) and verify the new disk
echo "- - -" > /sys/class/scsi_host/host0/scan
lsblk

# 2. Add the disk to the existing volume group and verify free space
vgextend vgdata /dev/sdc
vgs

# 3. Extend an existing LV (and file system), or create a new LV
lvextend -l +100%FREE -r /dev/vgdata/lvdata
# Or: lvcreate -n lvdata2 -L 20G vgdata
```

## Physical Volume, Volume Group, and Logical Volume Commands

```bash
# Physical Volumes
pvcreate /dev/sdb
pvresize /dev/sda2
pvs
pvdisplay
pvremove /dev/sdb

# Volume Groups
vgcreate vgdata /dev/sdb
vgextend vgdata /dev/sdc
vgs
vgdisplay
vgremove vgdata

# Logical Volumes
lvcreate -n lvdata -L 10G vgdata
lvcreate -n lvdata -l 50%FREE vgdata
lvcreate -n lvdata -l 100%FREE vgdata
lvextend -L +5G -r /dev/vgdata/lvdata
lvextend -l +100%FREE -r /dev/vgdata/lvdata
lvs
lvdisplay
lvremove /dev/vgdata/lvdata
```

### Logical Volume Resize Options

| Option | Description |
|---|---|
| `-r` | Automatically resize the file system after resizing the logical volume. |
| `-L` | Specify size in bytes, MB, GB, and so on. A `+` adds capacity; an absolute value sets total capacity. |
| `-l` | Specify extents or a percentage, such as `%FREE` or `%VG`. |

{: .tip }
> Use `-r` with `lvextend` to resize the file system automatically after increasing the logical volume.

## File System Types and Creation

| Type | Description |
|---|---|
| `xfs` | Default file system type in RHEL; can grow but cannot shrink. |
| `ext4` | Can grow and shrink; shrinking requires the file system to be unmounted. |
| `tmpfs` | Temporary file system stored in RAM. |
| `swap` | Virtual memory swap space. |

```bash
# Format as XFS, Ext4, or swap
mkfs.xfs /dev/vgdata/lvdata
mkfs.ext4 /dev/vgdata/lvdata
mkswap /dev/vgdata/lvswap
```

{: .warning }
> Formatting commands such as `mkfs.xfs`, `mkfs.ext4`, and `mkswap` overwrite existing file system metadata. Verify the target device before running them.

## Mounting and `/etc/fstab`

```bash
# Mount by device path or UUID, unmount, or mount all /etc/fstab entries
mount /dev/vgdata/lvdata /data
mount UUID="xxx" /data
umount /data
mount -a

# Activate swap and check swap usage
swapon /dev/vgdata/lvswap
free -h

# Verify /etc/fstab syntax and mounted file systems
findmnt --verify
df -h
```

### `/etc/fstab` Fields

| Field | Description |
|---|---|
| Device | Device name, UUID, or label. |
| Mount Point | Directory where the file system is mounted. |
| File System Type | `xfs`, `ext4`, `swap`, `tmpfs`, and so on. |
| Options | `defaults` or specific mount options. |
| Dump | `0` disables dump backup support; `1` enables it. |
| `fsck` Order | `0` disables checks; `1` is normally for the root file system; `2` is for other file systems. |

### Common Mount Options

| Option | Description |
|---|---|
| `defaults` | Use default mount options. |
| `ro` / `rw` | Mount read-only or read-write. |
| `noexec` | Prevent program execution from the file system. |
| `nosuid` | Ignore SUID and SGID bits. |
| `nodev` | Ignore device files. |
| `_netdev` | Wait for network availability; used for network file systems. |

### `/etc/fstab` Examples

```fstab
# Logical volume using device path
/dev/vgdata/lvdata /data xfs defaults 0 2

# Logical volume using UUID
UUID="xxx" /data xfs defaults 0 2

# Swap logical volume
/dev/vgdata/lvswap swap swap defaults 0 0

# Basic tmpfs mount and size-limited tmpfs mount
tmpfs /tmp tmpfs defaults 0 0
tmpfs /tmp tmpfs size=2G,mode=1777 0 0
```

{: .warning }
> Test `/etc/fstab` with `mount -a` before rebooting. An invalid entry can prevent normal boot or delay startup.