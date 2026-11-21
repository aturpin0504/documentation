---
layout: default
title: Boot Procedure and Troubleshooting
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
2. Find the line beginning with:

   ```text
   linux ($root)/vmlinuz
   ```

3. Add the required boot arguments to the end of that line.
4. Press `Ctrl-X` to boot with the modified arguments.

{: .note }
> Boot arguments entered through the GRUB menu are temporary. To make changes persistent, update the appropriate GRUB configuration source, such as `/etc/default/grub`, and regenerate GRUB configuration as required.

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

1. At the GRUB boot menu, press `e` and add `rd.break` to the end of the `linux` line.
2. Press `Ctrl-X` to boot into the `initramfs` break environment.
3. Remount the system root file system as read-write:

   ```bash
   mount -o remount,rw /sysroot
   ```

4. Change root into the installed system image:

   ```bash
   chroot /sysroot
   ```

5. Set a new root password:

   ```bash
   passwd
   ```

6. Load the SELinux policy:

   ```bash
   load_policy -i
   ```

7. Correct the SELinux context for `/etc/shadow`:

   ```bash
   chcon -t shadow_t /etc/shadow
   ```

8. Exit the `chroot` environment:

   ```bash
   exit
   ```

9. Exit the `initramfs` environment and continue boot:

   ```bash
   exit
   ```

### Alternative SELinux Recovery Method

Instead of manually loading policy and correcting the context, create an autorelabel marker:

```bash
touch /.autorelabel
```

The system performs a full SELinux relabel during the next boot.

## Troubleshooting `/etc/fstab` Issues

### Common Symptoms

```text
Give root password for maintenance
```

Or the system boots into emergency mode.

### Common Causes

- A file system check (`fsck`) failed.
- A device listed in `/etc/fstab` cannot be found.
- An invalid mount option or file system type is configured.
- A UUID or device path has changed.

### Recovery Steps

```bash
# Review boot errors and the current boot journal
journalctl -xb

# Remount the root file system as read-write
mount -o remount,rw /

# Correct /etc/fstab entries
vi /etc/fstab

# Reboot after corrections
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

## Key Points

- Password recovery using `rd.break` is a common administrative recovery procedure.
- Remove `rhgb` and `quiet` from temporary GRUB boot arguments to view detailed boot messages.
- Correct SELinux labels after password recovery.
- `/etc/fstab` errors are a common cause of emergency-mode boots.
- Boot arguments changed interactively in GRUB are not persistent.
- `emergency.target` is more minimal than `rescue.target`.