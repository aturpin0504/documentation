---
layout: default
title: Advanced Recovery
parent: Quick Reference
grand_parent: RHEL
nav_order: 21
---

# Rescue Boot and GRUB Recovery

{: .note }
> These procedures are uncommon in daily operations and extremely rare in VM environments. Keep them for reference, but they are unlikely to be needed regularly.

## Rescue Boot Options

| Option | Purpose |
|---|---|
| **Rescue a RHEL System** | Most flexible rescue option. |
| **Boot from Local Drive** | Least intrusive option; attempts to use the local boot loader. |
| **Run Memory Test** | Tests memory and identifies bad memory. |

### Rescue System Commands

```bash
# Make the installed system image the root environment
chroot /mnt/sysimage

# Exit the chroot environment and reboot
exit
reboot
```

## Reinstalling GRUB 2

### From a Bootable System

```bash
# Physical server, VMware, or VirtualBox
grub2-install /dev/sda

# KVM virtual machine
grub2-install /dev/vda
```

### From Rescue Mode

```bash
# Change root into the installed system environment
chroot /mnt/sysimage

# Install GRUB to the boot disk
grub2-install /dev/sda

# Exit the chroot environment and reboot
exit
reboot
```

## Making GRUB Changes Persistent

```bash
# Edit GRUB configuration defaults
vi /etc/default/grub

# Generate GRUB configuration for BIOS systems
grub2-mkconfig -o /boot/grub2/grub.cfg

# Generate GRUB configuration for UEFI systems
grub2-mkconfig -o /boot/efi/EFI/redhat/grub.cfg
```

## Re-Creating `initramfs`

```bash
# Recreate initramfs, overwriting the existing image
dracut --force

# Create a new initramfs without overwriting an existing image
dracut
```

### Dracut Configuration Files

| File or Directory | Purpose |
|---|---|
| `/etc/dracut.conf` | Main Dracut configuration file. |
| `/etc/dracut.conf.d/` | Directory for custom Dracut configuration files. |
| `/usr/lib/dracut/dracut.conf.d/` | System default Dracut configuration files. |

## Key Terms

| Term | Definition |
|---|---|
| GRUB | Grand Unified Bootloader version 2. |
| `initramfs` | Initial RAM file system used during boot. |
| `dracut` | Tool used to create `initramfs` images. |
| `chroot` | Change-root environment that makes another directory tree appear as `/`. |
| `rd.break` | Boot argument that breaks into the `initramfs` environment for recovery. |