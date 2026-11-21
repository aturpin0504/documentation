---
layout: default
title: Kernel Management
parent: Quick Reference
grand_parent: RHEL
nav_order: 19
---

# Kernel Information and Management

## Kernel and System Information

```bash
# Show running kernel version and all system information
uname -r
uname -a

# Show detailed system information, including virtualization
hostnamectl status

# Show RHEL version
cat /etc/redhat-release
```

### Memory Aids

- `uname` — Unix name; displays system identity information.
- `dmesg` — Display messages; shows kernel ring-buffer messages.
- `udev` — Userspace device manager.

## Kernel Messages

```bash
# View recent kernel messages (or with a pager)
dmesg
dmesg | less

# View kernel messages from the journal
journalctl -k

# Check whether the kernel is tainted (proprietary or unsupported modules loaded)
dmesg | grep -i taint
```

## Kernel Modules

```bash
# List loaded kernel modules and show PCI devices with drivers
lsmod
lspci -k

# Show module details and available parameters (look for parm: lines)
modinfo module_name

# Load, load with a parameter, or unload a module
modprobe module_name
modprobe cdrom debug=1
modprobe -r module_name
```

To make module parameters persistent, create a file under `/etc/modprobe.d/`:

```ini
# /etc/modprobe.d/cdrom.conf
options cdrom debug=1
```

{: .tip }
> Use `modprobe` rather than `insmod` or `rmmod`. `modprobe` handles module dependencies automatically.

## Kernel Updates and Reboot Checks

Kernel updates install alongside existing kernels, allowing rollback to an earlier kernel if necessary.

```bash
# Install a kernel update and view installed kernels
yum upgrade kernel
rpm -q kernel

# Check whether a reboot is required (install yum-utils if needed)
needs-restarting -r
yum install yum-utils
```

### Check Without `needs-restarting`

Compare the running kernel (`uname -r`) with the newest installed kernel (`rpm -q kernel`). If they do not match, a reboot is required:

```bash
uname -r
rpm -q kernel
```

### After a Kernel Update

```bash
# Reboot to load the newest installed kernel (GRUB selects it by default)
reboot

# Verify the running kernel matches the newest installed kernel
uname -r
rpm -q kernel | tail -1
```

## Booting an Older Kernel

1. At boot, press any key when the GRUB countdown appears.
2. Use the arrow keys to select the desired older kernel.
3. Press `Enter` to boot the selected kernel.

## Key Points

- RHEL commonly retains multiple kernel versions in `/boot`; verify the configured retention policy for the system.
- Many kernel-management commands read from `/proc` and present the information in a more usable format; prefer those commands over manually reading raw `/proc` files.