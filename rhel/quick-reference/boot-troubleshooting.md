---
layout: default
title: Boot Troubleshooting
parent: Quick Reference
grand_parent: RHEL
nav_order: 21
---

# Boot Procedure and Troubleshooting

## Boot Components Overview

| Component | Purpose |
|---|---|
| BIOS or UEFI firmware | Initializes hardware and selects a boot device. |
| GRUB 2 | Boot loader that presents boot entries and loads the selected kernel and `initramfs`. |
| Kernel | Core of the operating system; initializes CPU, memory, drivers, and core hardware functions. |
| `initramfs` | Temporary root filesystem loaded into memory during early boot. Contains the tools, drivers, and modules needed to locate and mount the real root filesystem. |
| `systemd` | First user-space process on the real root filesystem; starts services and reaches the default target. |

### GRUB 2, UEFI, and the Boot Sector

**GRUB 2** is the boot loader used by RHEL. It loads the selected Linux kernel and its associated `initramfs` image into memory, then transfers control to the kernel.

On a traditional BIOS system, boot code begins in the disk's boot sector or Master Boot Record (MBR).

On a UEFI system:

- UEFI firmware reads boot files from the EFI System Partition (ESP).
- The firmware starts an EFI boot-loader application.
- Secure Boot can validate signatures of boot components before they run.
- The traditional disk boot sector is not the normal boot path.

{: .note }
> On UEFI systems with Secure Boot enabled, do not use `grub2-install` as a routine repair action. Use the supported RHEL package-reinstallation procedure for GRUB EFI and shim components.

## Boot Procedure and Troubleshooting Reference

| Phase | What Happens | Common Problem | Primary Check or Recovery Method |
|---|---|---|---|
| POST | Firmware initializes CPU, memory, and basic hardware. | Hardware fault or firmware configuration issue. | Review firmware messages, device health, and BIOS/UEFI configuration. |
| Boot Device Selection | Firmware selects a bootable device. | Incorrect boot order or unavailable boot disk. | Verify boot order and storage-device availability in firmware. |
| GRUB 2 | Boot loader displays entries and loads the selected kernel and `initramfs`. | Missing, damaged, or misconfigured boot entry. | Use the GRUB menu, review GRUB configuration, or use rescue media. |
| Kernel Load | Kernel initializes memory, CPU, and built-in drivers. | Kernel panic, missing driver, or invalid kernel argument. | Remove temporary arguments, select an earlier kernel, or review `journalctl -k`. |
| `initramfs` | Temporary root filesystem loads early drivers and locates the real root filesystem. | Missing storage, LVM, encryption, or filesystem driver. | Use the dracut shell, review boot arguments, or rebuild the `initramfs`. |
| `initrd.target` | The `systemd` instance in the `initramfs` mounts the real root filesystem and prepares to switch root. | Real root filesystem cannot be located or mounted. | Review storage, LVM, encryption, filesystem, and `fstab` configuration. |
| Root File System Mount | The real root filesystem mounts. | Invalid `/etc/fstab`, failed filesystem check, or unavailable device. | Boot to emergency mode, remount root read/write, and correct `/etc/fstab`. |
| `systemd` Startup | `systemd` starts services and mounts required filesystems. | Failed service, failed mount, or dependency issue. | Use `journalctl -xb`, `systemctl --failed`, and target recovery options. |
| Default Target | The system reaches `multi-user.target`, `graphical.target`, or another configured target. | Incorrect default target or failed units. | Check `systemctl get-default` and inspect failed units. |

## GRUB 2 Boot Menu Access

| Key | Action |
|---|---|
| `e` | Edit the selected boot entry temporarily. |
| `c` | Enter the GRUB command prompt. |
| `Ctrl-X` | Boot using the temporarily edited arguments. |
| `Esc` | Return to the previous GRUB screen, when supported. |

{: .note }
> GRUB arguments entered through the boot menu apply only to the current boot. To make a boot-argument change persistent, use `grubby` or modify the approved GRUB configuration source and regenerate the configuration.

## Editing GRUB Boot Arguments Temporarily

1. Reboot the system.
2. At the GRUB 2 menu, select the desired boot entry.
3. Press `e`.
4. Locate the kernel command line. On UEFI systems, this commonly begins with `linuxefi`.
5. Add or remove the required boot arguments at the end of the line.
6. Press `Ctrl-X` to boot with the temporary arguments.

Example temporary troubleshooting arguments:

```text
rd.break
systemd.unit=emergency.target
systemd.unit=rescue.target
```

## Common GRUB Kernel Parameters

The `GRUB_CMDLINE_LINUX` value in `/etc/default/grub` contains kernel arguments added to Linux boot entries.

Example:

```ini
GRUB_CMDLINE_LINUX="rd.lvm.lv=rhel/root rd.lvm.lv=rhel/swap rhgb quiet"
```

| Parameter | Purpose |
|---|---|
| `rhgb` | Enables the Red Hat graphical boot display. |
| `quiet` | Reduces boot messages displayed on the console. |
| `rd.break` | Stops boot processing in the `initramfs` before the real root filesystem is mounted normally. |
| `systemd.unit=rescue.target` | Boots to the rescue target. |
| `systemd.unit=emergency.target` | Boots to the minimal emergency target. |
| `rd.lvm.lv=<vg>/<lv>` | Identifies a required logical volume during early boot. |
| `rd.luks.uuid=<uuid>` | Identifies an encrypted device required during early boot. |
| `crashkernel=auto` | Reserves memory for `kdump` when configured. |

{: .warning }
> Do not casually remove storage, encryption, LVM, or root-device arguments such as `rd.lvm.lv=`, `rd.luks.uuid=`, or `root=`. Removing required arguments can prevent the system from locating or mounting the root filesystem.

### Modify Kernel Arguments Persistently

Use `grubby` to add or remove an argument for all installed kernel entries:

```bash
# Display all boot entries and kernel arguments
grubby --info=ALL

# Add a kernel argument to all boot entries
grubby --update-kernel=ALL --args="<kernel-argument>"

# Remove a kernel argument from all boot entries
grubby --update-kernel=ALL --remove-args="<kernel-argument>"

# Display the default boot kernel
grubby --default-kernel
```

For configuration managed through `/etc/default/grub`, update the `GRUB_CMDLINE_LINUX` line and regenerate the GRUB configuration.

```bash
# Edit the GRUB defaults file
vi /etc/default/grub

# Regenerate the RHEL GRUB configuration
grub2-mkconfig -o /boot/grub2/grub.cfg --update-bls-cmdline
```

{: .warning }
> Do not manually edit generated `grub.cfg` files. Changes can be overwritten when GRUB configuration is regenerated or when kernel packages are updated.

### GRUB Default Entry and GRUB Authentication

`GRUB_DEFAULT` selects the default boot entry. It is not a username.

```bash
# Display the default boot target selected by GRUB
grubby --default-kernel

# Display all available boot entries
grubby --info=ALL
```

GRUB password protection is separate from the Linux root password. A GRUB password can protect boot-entry editing and, depending on configuration, booting selected entries.

{: .note }
> If GRUB is configured to require authentication before editing entries, you must provide the configured GRUB username and password before pressing `e` to modify boot arguments. Follow the approved organizational baseline for GRUB password configuration, boot-entry access, and related STIG requirements.

### When GRUB Prompts for Authentication

A GRUB username and password are requested only for protected GRUB actions. They are separate from normal Linux user credentials.

| Action | Authentication Normally Required? |
|---|---|
| Boot the default, unrestricted menu entry | No |
| Select an unrestricted boot entry | No |
| Press `e` to edit boot arguments | Yes, when GRUB edit protection is configured |
| Press `c` to open the GRUB command prompt | Yes, when GRUB command-line access is protected |
| Boot an entry explicitly configured as restricted | Yes |

When authentication is required, GRUB displays prompts similar to:

```text
Enter username:
Enter password:
```

Use the configured **GRUB username and password**.

{: .note }
> A standard `grub2-setpassword` configuration generally protects boot-entry editing without requiring authentication to boot existing unrestricted entries. If individual boot entries are configured to require authentication, GRUB can also prompt before allowing those entries to boot.

## `initramfs` and `initrd.target`

An **initial RAM filesystem** (`initramfs`) is a temporary root filesystem loaded into memory during boot.

It commonly contains:

- Storage-controller drivers.
- Disk and filesystem modules.
- Device-mapper and LVM tools.
- Encryption tools and modules.
- Network drivers when network-based storage is required.
- The `dracut` boot environment.
- A minimal `systemd` environment.

The `initramfs` does not necessarily contain every installed kernel module. It includes the drivers and tools needed early in the boot process to locate and mount the real root filesystem.

`initrd.target` is the default `systemd` target while the system is operating inside the `initramfs`. It prepares the real root filesystem and transitions control to the normal `systemd` instance on that filesystem.

## Systemd Targets During Boot and Recovery

A **target** is a `systemd` unit that groups other units into a defined operational state. Targets are similar in purpose to SysV init runlevels.

Keep the full target-management reference in the Systemd guide. This section covers targets relevant to boot and recovery.

| Target | Description |
|---|---|
| `emergency.target` | Minimal recovery environment with only essential services. |
| `rescue.target` | Recovery environment with more services than emergency mode. |
| `multi-user.target` | Multi-user, non-graphical operating mode; common server default. |
| `graphical.target` | Multi-user operating mode with a graphical interface. |
| `reboot.target` | Stops services and reboots the system. |
| `poweroff.target` | Stops services and powers off the system. |
| `default.target` | Alias that identifies the target systemd starts during a normal boot. |

### Target Dependencies

| Directive | Purpose |
|---|---|
| `Wants=` | Weak dependency. Starts the listed unit when possible, but does not fail the requesting unit if the dependency fails. |
| `Requires=` | Strong dependency. The listed unit is required; failure can prevent the requesting unit from starting successfully. |
| `After=` | Ordering dependency. Starts this unit after the listed unit, but does not start the listed unit by itself. |
| `Before=` | Ordering dependency. Starts this unit before the listed unit. |
| `Conflicts=` | Negative dependency. Starting one unit stops the conflicting unit. |
| `AllowIsolate=yes` | Allows a target to be selected with `systemctl isolate`. |

### Target Commands

```bash
# Show the default target
systemctl get-default

# Set the default target for future boots
systemctl set-default multi-user.target
systemctl set-default graphical.target

# List currently loaded target units
systemctl list-units --type=target

# List all installed target-unit files
systemctl list-unit-files --type=target

# Show units and dependencies associated with a target
systemctl list-dependencies multi-user.target

# Switch immediately to another target
systemctl isolate rescue.target
```

### Enabling and Disabling Units

`systemctl enable` and `systemctl disable` affect whether a unit starts automatically through configured dependencies or installation links.

```bash
# Enable a service for future boots
systemctl enable <service>

# Disable a service from starting automatically
systemctl disable <service>

# Enable a service and start it immediately
systemctl enable --now <service>

# Disable a service and stop it immediately
systemctl disable --now <service>
```

| Command | Effect |
|---|---|
| `systemctl enable <unit>` | Configures the unit to start automatically when its installation target is reached. Does not start it immediately. |
| `systemctl disable <unit>` | Removes automatic-start configuration. Does not stop a currently running unit. |
| `systemctl enable --now <unit>` | Enables the unit for future boots and starts it now. |
| `systemctl disable --now <unit>` | Disables the unit for future boots and stops it now. |
| `systemctl isolate <target>` | Starts the selected target and stops units not required by that target. |

{: .warning }
> `systemctl isolate` can stop active services, network connectivity, graphical sessions, mounts, or applications that are not part of the selected target. Use it only when you understand the recovery impact.

## Troubleshooting Boot Options

| Argument | Purpose |
|---|---|
| `rd.break` | Stops during `initramfs` processing; used for authorized root-password recovery and early-boot troubleshooting. |
| `systemd.unit=emergency.target` | Starts the minimal emergency environment. |
| `systemd.unit=rescue.target` | Starts a more complete recovery environment. |
| Remove `rhgb quiet` | Displays detailed boot messages rather than suppressing them. |

## Authorized Root Password Recovery Procedure

{: .warning }
> Perform password-recovery actions only on systems you are authorized to administer. Changing the root password affects system access and may require coordination with the system owner or security staff.

Use the `rd.break` procedure for RHEL root-password recovery.

1. Reboot the system.
2. At the GRUB menu, select the desired boot entry and press `e`.
3. Add `rd.break` to the end of the kernel command line.
4. Press `Ctrl-X` to boot into the `initramfs` break environment.
5. Remount the installed root filesystem as read-write.
6. Change root into the installed system.
7. Reset the password.
8. Create the SELinux relabel marker.
9. Exit twice to resume boot.

```bash
# Remount the installed root filesystem as read-write
mount -o remount,rw /sysroot

# Enter the installed system
chroot /sysroot

# Set the new root password
passwd

# Request a full SELinux relabel on the next boot
touch /.autorelabel

# Exit the chroot environment
exit

# Exit the initramfs break environment and resume boot
exit
```

{: .note }
> The correct relabel command is `touch /.autorelabel` with no space between `/` and `.autorelabel`. The next boot can take significantly longer while SELinux relabels the filesystem.

### About `init=/bin/bash`

Booting with `init=/bin/bash` can start a root shell on some systems, but it bypasses the standard RHEL boot flow and is not the preferred recovery method for this guide.

The `rd.break` procedure is preferred because it:

- Uses the RHEL-supported `initramfs` recovery workflow.
- Provides access to the installed root filesystem through `/sysroot`.
- Uses `chroot` before changing the password.
- Includes the SELinux relabel step.

## Troubleshooting `/etc/fstab` Issues

If `/etc/fstab` contains an invalid entry, references a missing device, or requires a failed filesystem check, the system can enter emergency mode.

```bash
# Review the current boot journal and boot errors
journalctl -xb

# Remount the root file system as read-write, if needed
mount -o remount,rw /

# Correct the fstab configuration
vi /etc/fstab

# Test fstab entries without rebooting
mount -a

# Reboot after correcting the issue
reboot
```

{: .tip }
> Use stable UUID or LABEL identifiers in `/etc/fstab` where appropriate. Test changes with `mount -a` before rebooting whenever possible.

## Rebuild an `initramfs` Image with Dracut

Use `dracut` when the `initramfs` is missing, damaged, or lacks a required early-boot driver or configuration.

```bash
# Show the running kernel
uname -r

# Rebuild the initramfs for the running kernel
dracut --force --kver "$(uname -r)"

# Verify that the initramfs image exists
ls -lh /boot/initramfs-"$(uname -r)".img
```

To rebuild an `initramfs` for a specific installed kernel:

```bash
# List installed kernel-core packages
rpm -q kernel-core

# Rebuild the initramfs for a specific kernel release
dracut --force --kver <kernel-release>

# Verify the resulting image
ls -lh /boot/initramfs-<kernel-release>.img
```

{: .warning }
> Rebuild an `initramfs` only after identifying the affected kernel version and the underlying issue. For example, rebuilding may be appropriate after correcting a missing storage-driver, LVM, encryption, or dracut configuration problem.

## Reinstall GRUB 2 on a UEFI System

{: .warning }
> Use GRUB repair only when a GRUB, shim, EFI boot-loader, or related package problem has been identified. On UEFI systems with Secure Boot enabled, do not use `grub2-install` as a general repair command.

First, confirm that the system is using UEFI:

```bash
# UEFI systems expose this directory
ls /sys/firmware/efi

# Display installed GRUB and shim packages
rpm -qa | grep -E '^(grub2|shim)'
```

For a typical x86_64 UEFI system, reinstall the installed GRUB EFI, shim, and GRUB utility packages:

```bash
dnf reinstall -y \
  grub2-efi-x64 \
  shim-x64 \
  grub2-tools \
  grub2-common
```

Regenerate the GRUB configuration after correcting the boot-loader packages or approved GRUB configuration:

```bash
grub2-mkconfig -o /boot/grub2/grub.cfg --update-bls-cmdline
```

Then reboot and verify the system starts successfully.

```bash
reboot
```

{: .note }
> Package names can vary by RHEL release and system architecture. Use `rpm -qa | grep -E '^(grub2|shim)'` to identify the installed package names before reinstalling packages.

## Boot Diagnostics

```bash
# Review the current boot journal
journalctl -b

# Review boot errors
journalctl -b -p err

# Review failed systemd units
systemctl --failed

# Review kernel messages from the current boot
journalctl -k -b

# Review available boot entries
grubby --info=ALL

# Display the configured default target
systemctl get-default

# Display the default GRUB boot kernel
grubby --default-kernel
```

## Key Points

- Keep full target management in the Systemd guide; retain boot-recovery target information in this guide.
- GRUB 2 loads the kernel and `initramfs`; UEFI firmware starts the EFI boot chain.
- The `initramfs` contains early-boot tools and drivers required to locate and mount the real root filesystem.
- `initrd.target` is the target used while systemd operates inside the `initramfs`.
- Use `grubby` for targeted kernel-command-line changes and avoid manually editing generated GRUB configuration files.
- Use `rd.break` for authorized root-password recovery.
- A GRUB password can require authentication before editing boot entries.
- Use `dracut --force --kver <kernel-release>` to rebuild an affected `initramfs`.
- On UEFI systems, repair GRUB through supported package reinstallation rather than using `grub2-install` as a routine action.
