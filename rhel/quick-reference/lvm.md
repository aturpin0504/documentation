---
layout: default
title: Understanding Logical Volume Manager (LVM)
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

## Why Use LVM?

- Default storage configuration for many RHEL 8+ installations.
- Supports flexible volume resizing, often without downtime.
- Allows storage capacity to be added more easily.
- Commonly used in VMware environments.
- Supports creating multiple logical volumes from a shared storage pool.

## Viewing Storage

| Task | Command |
|---|---|
| List block devices and hierarchy | `lsblk` |
| Show physical volume summary | `pvs` |
| Show volume group summary | `vgs` |
| Show logical volume summary | `lvs` |
| Display UUIDs for `/etc/fstab` | `blkid` |
| Show file system disk usage | `df -h` |

### Device References for `/etc/fstab`

For LVM-backed file systems, use either:

- **Logical volume device path:**

  ```text
  /dev/vgname/lvname
  ```

- **UUID:**

  ```text
  UUID="xxx"
  ```

Examples:

```text
/dev/vgdata/lvdata
UUID="1234-5678-90ab-cdef"
```

Use `lvs` to identify volume group and logical volume names, or use `blkid` to identify file system UUIDs.

## Creating New LVM Storage

The following example creates a new volume group and logical volume using a newly attached disk.

```bash
# 1. View the new disk
lsblk

# 2. Create a volume group
vgcreate vgdata /dev/sdb

# 3. Create a 10 GB logical volume
lvcreate -n lvdata -L 10G vgdata

# 4. Create an XFS file system
mkfs.xfs /dev/vgdata/lvdata

# 5. Create the mount point
mkdir /data

# 6. Add the file system to /etc/fstab
# /dev/vgdata/lvdata /data xfs defaults 0 2

# 7. Mount all /etc/fstab entries
mount -a

# 8. Verify the mount
df -h /data
```

Example `/etc/fstab` entry:

```fstab
/dev/vgdata/lvdata /data xfs defaults 0 2
```

## VMware: Extending an Existing VMDK

### Scenario

A VMware virtual disk, `/dev/sda`, was expanded from 50 GB to 100 GB in vSphere while the virtual machine remains online.

```bash
# 1. Rescan the disk for the new size
echo 1 > /sys/class/block/sda/device/rescan

# 2. Verify the new disk size
lsblk

# 3. Resize the physical volume
pvresize /dev/sda2

# 4. Confirm the volume group has free space
vgs

# 5. Extend the logical volume and file system
lvextend -l +100%FREE -r /dev/vgdata/lvdata

# 6. Verify the expanded file system
df -h /data
```

## VMware: Adding a New Disk to an Existing Volume Group

### Scenario

A new disk, `/dev/sdc`, was added to a running virtual machine and must be added to the existing `vgdata` volume group.

```bash
# 1. Rescan the SCSI bus, if needed
echo "- - -" > /sys/class/scsi_host/host0/scan

# 2. Verify the new disk is visible
lsblk

# 3. Add the disk to the existing volume group
vgextend vgdata /dev/sdc

# 4. Verify that the volume group has free space
vgs

# 5. Extend an existing logical volume and file system
lvextend -l +100%FREE -r /dev/vgdata/lvdata
```

Alternatively, create a new logical volume:

```bash
lvcreate -n lvdata2 -L 20G vgdata
```

## Volume Group Commands

| Task | Command |
|---|---|
| Create a volume group on a disk | `vgcreate vgdata /dev/sdb` |
| Add a disk or PV to an existing volume group | `vgextend vgdata /dev/sdc` |
| Show volume group summary | `vgs` |
| Show detailed volume group information | `vgdisplay` |
| Remove a volume group | `vgremove vgdata` |

## Logical Volume Commands

| Task | Command |
|---|---|
| Create a 10 GB logical volume | `lvcreate -n lvdata -L 10G vgdata` |
| Create an LV using 50% of free space | `lvcreate -n lvdata -l 50%FREE vgdata` |
| Create an LV using all free space | `lvcreate -n lvdata -l 100%FREE vgdata` |
| Extend an LV by 5 GB and resize the file system | `lvextend -L +5G -r /dev/vgdata/lvdata` |
| Extend an LV to use all free space | `lvextend -l +100%FREE -r /dev/vgdata/lvdata` |
| Show logical volume summary | `lvs` |
| Show detailed logical volume information | `lvdisplay` |
| Remove a logical volume | `lvremove /dev/vgdata/lvdata` |

### Logical Volume Resize Options

| Option | Description |
|---|---|
| `-r` | Automatically resize the file system after resizing the logical volume. |
| `-L` | Specify size in bytes, MB, GB, and so on. A `+` adds capacity; an absolute value sets total capacity. |
| `-l` | Specify extents or a percentage, such as `%FREE` or `%VG`. |

{: .tip }
> Use `-r` with `lvextend` to resize the file system automatically after increasing the logical volume.

## Physical Volume Commands

| Task | Command |
|---|---|
| Create a physical volume | `pvcreate /dev/sdb` |
| Resize a PV after disk expansion | `pvresize /dev/sda2` |
| Show physical volume summary | `pvs` |
| Show detailed physical volume information | `pvdisplay` |
| Remove an LVM PV signature | `pvremove /dev/sdb` |

## File System Types

| Type | Description |
|---|---|
| `xfs` | Default file system type in RHEL 8; can grow but cannot shrink. |
| `ext4` | Can grow and shrink; shrinking requires the file system to be unmounted. |
| `tmpfs` | Temporary file system stored in RAM. |
| `swap` | Virtual memory swap space. |

## Creating File Systems

| Task | Command |
|---|---|
| Format as XFS | `mkfs.xfs /dev/vgdata/lvdata` |
| Format as Ext4 | `mkfs.ext4 /dev/vgdata/lvdata` |
| Format as swap | `mkswap /dev/vgdata/lvswap` |

{: .warning }
> Formatting commands such as `mkfs.xfs`, `mkfs.ext4`, and `mkswap` overwrite existing file system metadata. Verify the target device before running them.

## Mounting File Systems

| Task | Command |
|---|---|
| Mount by device path | `mount /dev/vgdata/lvdata /data` |
| Mount by UUID | `mount UUID="xxx" /data` |
| Unmount by mount point | `umount /data` |
| Mount all `/etc/fstab` entries | `mount -a` |
| Activate swap | `swapon /dev/vgdata/lvswap` |
| Show swap usage | `free -h` |

## `/etc/fstab` Configuration

### Fields

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

# Basic tmpfs mount
tmpfs /tmp tmpfs defaults 0 0

# tmpfs mount with a 2 GB size limit and sticky-bit permissions
tmpfs /tmp tmpfs size=2G,mode=1777 0 0
```

## Testing `/etc/fstab`

| Task | Command |
|---|---|
| Test all `/etc/fstab` entries | `mount -a` |
| Verify mounted file systems | `df -h` |
| Verify `/etc/fstab` syntax | `findmnt --verify` |

{: .warning }
> Test `/etc/fstab` with `mount -a` before rebooting. An invalid entry can prevent normal boot or delay startup.

## Best Practices

- Create volume groups directly on whole disks with `vgcreate` when partitioning is not required.
- After expanding a VMware VMDK, use `pvresize` on the applicable physical volume.
- Use `vgextend` to add new disks to an existing volume group.
- Use `lvextend -r` to extend both the logical volume and its file system.
- Test `/etc/fstab` with `mount -a` before rebooting.
- XFS file systems can grow but cannot be shrunk.
- Use `lvs` to identify LVM device paths and `blkid` to retrieve UUIDs.
- A default RHEL LVM installation commonly uses LVM for `/` and swap, but not for `/boot`.