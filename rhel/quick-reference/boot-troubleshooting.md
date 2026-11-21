---
layout: default
title: Boot Troubleshooting
parent: Quick Reference
grand_parent: RHEL
nav_order: 20
---

# Boot Procedure and Troubleshooting

## Boot Procedure Phases

| Phase | Description |
|---|---|
| POST | Power-On Self-Test; hardware initialization. |
| Boot Device Selection | BIOS or UEFI selects the boot device. |
| GRUB 2 Load | GRUB 2 boot loader loads. |
| Kernel Load | Kernel and `initramfs` load. |
| Init Start | `/sbin/init` (`systemd`) starts. |
| `initrd.target` | Initial RAM disk processing occurs. |
| Root File System Mount | Root file system is mounted. |
| Default Target | System reaches its configured operational target. |

## GRUB 2 Boot Menu Access

| Key | Action |
|---|---|
| `e` | Edit boot commands. |
| `c` | Enter the GRUB command prompt. |
| `Ctrl-X` | Boot using the current arguments. |

## Editing GRUB Boot Arguments

1. Boot the system and press `e` at the GRUB menu.
2. Find the line beginning with `linux ($root)/vmlinuz`.
3. Add the required boot arguments to the end of that line.
4. Press `Ctrl-X` to boot with the modified arguments.

{: .note }
> Boot arguments entered through the GRUB menu are temporary. To make changes persistent, update `/etc/default/grub` and regenerate the GRUB configuration.

## Troubleshooting Boot Options

| Argument | Purpose |
|---|---|
| `rd.break` | Stop during `initramfs` processing; commonly used for authorized password recovery. |
| `init=/bin/bash` | Start a shell after kernel loading. |
| `systemd.unit=emergency.target` | Start the minimal emergency environment; root authentication is required. |
| `systemd.unit=rescue.target` | Start a more complete recovery environment; root authentication is required. |
| Remove `rhgb quiet` | Display detailed boot messages rather than suppressing them. |

## Authorized Root Password Recovery Procedure

{: .warning }
> Perform password-recovery actions only on systems you are authorized to administer. Changing the root password affects system access and may require coordination with the system owner or security staff.

1. At the GRUB boot menu, press `e`, add `rd.break` to the end of the `linux` line, and press `Ctrl-X`.
2. Remount `/sysroot`, `chroot` into it, reset the password, restore SELinux context, and exit:

```bash
mount -o remount,rw /sysroot
chroot /sysroot
passwd
load_policy -i
chcon -t shadow_t /etc/shadow
exit
exit
```

Alternatively, instead of `load_policy -i` and `chcon`, run `touch /.autorelabel` before exiting to trigger a full SELinux relabel on the next boot.

## Troubleshooting `/etc/fstab` Issues

If `/etc/fstab` has an invalid entry, missing device, or failed `fsck`, the system may drop to `Give root password for maintenance` (emergency mode).

```bash
# Review boot errors and the current boot journal
journalctl -xb

# Remount the root file system as read-write
mount -o remount,rw /

# Correct /etc/fstab entries and reboot
vi /etc/fstab
reboot
```

{: .tip }
> Test `/etc/fstab` changes with `mount -a` before rebooting whenever possible.

## Boot Phase Troubleshooting Reference

| Boot Phase | Primary Configuration | Common Fix Method |
|---|---|---|
| POST | Hardware configuration; firmware setup such as `F2`, `Esc`, or `F10` | Replace or correct hardware configuration. |
| Boot Device | BIOS or UEFI boot-order configuration | Correct boot order, replace hardware, or use rescue media. |
| Boot Loader | `grub2-install`, `/etc/default/grub` | Use GRUB prompt or rebuild configuration with `grub2-mkconfig`. |
| Kernel | `/etc/default/grub`, `/etc/dracut.conf` | Use GRUB boot arguments or rebuild configuration with `grub2-mkconfig`. |
| Init Start | Compiled into the `initramfs` | Use `init=` or `rd.break` boot arguments. |
| Root File System | `/etc/fstab` | Repair invalid `/etc/fstab` entries. |
| Default Target | `systemctl set-default` | Use a `rescue.target` boot argument or correct the default target. |