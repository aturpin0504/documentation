---
layout: default
title: Kernel Information and Management
parent: Quick Reference
grand_parent: RHEL
nav_order: 19
---

# Kernel Information and Management

## Kernel and System Information

| Task | Command |
|---|---|
| Show running kernel version | `uname -r` |
| Show all system information | `uname -a` |
| Show detailed system information, including virtualization | `hostnamectl status` |
| Show RHEL version | `cat /etc/redhat-release` |

### Memory Aids

- `uname` — Unix name; displays system identity information.
- `dmesg` — Display messages; shows kernel ring-buffer messages.
- `udev` — Userspace device manager.

## Kernel Messages

| Task | Command |
|---|---|
| View recent kernel messages | `dmesg` |
| View kernel messages with a pager | <code>dmesg &#124; less</code> |
| View kernel messages from the journal | `journalctl -k` |
| Check whether the kernel is tainted | <code>dmesg &#124; grep -i taint</code> |

Examples:

```bash
dmesg
dmesg | less
journalctl -k
dmesg | grep -i taint
```

## Kernel Modules

| Task | Command |
|---|---|
| List loaded kernel modules | `lsmod` |
| Show module details and parameters | `modinfo module_name` |
| Load a module | `modprobe module_name` |
| Load a module with a parameter | `modprobe cdrom debug=1` |
| Unload a module | `modprobe -r module_name` |
| Show PCI devices and associated drivers | `lspci -k` |

### Module Parameters

- Use `modinfo module_name` to view available parameters in `parm:` lines.
- Load a module with parameters:

  ```bash
  modprobe cdrom debug=1
  ```

- To make module parameters persistent, create a file under `/etc/modprobe.d/`:

  ```ini
  # /etc/modprobe.d/cdrom.conf
  options cdrom debug=1
  ```

{: .tip }
> Use `modprobe` rather than `insmod` or `rmmod`. `modprobe` handles module dependencies automatically.

## Kernel Updates and Reboot Checks

### Install a Kernel Update

Kernel updates install alongside existing kernels, allowing rollback to an earlier kernel if necessary.

```bash
yum upgrade kernel
```

### View Installed Kernels

```bash
rpm -q kernel
```

### Check Whether a Reboot Is Required

```bash
needs-restarting -r
```

Install the tool if needed:

```bash
yum install yum-utils
```

Example output:

```text
Core libraries or services have been updated since boot-up:
  * kernel

Reboot is required to fully utilize these updates.
```

### Check Without `needs-restarting`

Compare the currently running kernel with the newest installed kernel.

```bash
# Show the running kernel
uname -r

# List installed kernels
rpm -q kernel
```

If the running kernel matches the newest installed kernel, a reboot is generally not required for the kernel update.

If the running kernel does not match the newest installed kernel, reboot to load the newest kernel.

Example:

```text
uname -r output:
4.18.0-372.9.1.el8.x86_64

rpm -q kernel output:
kernel-4.18.0-372.9.1.el8.x86_64
kernel-4.18.0-425.3.1.el8.x86_64
```

Result: A reboot is required to load `4.18.0-425.3.1.el8.x86_64`.

## After a Kernel Update

```bash
# Reboot the system
reboot

# Verify the running kernel after reboot
uname -r

# Display the newest installed kernel
rpm -q kernel | tail -1
```

After rebooting, GRUB normally selects the newest installed kernel by default.

## Booting an Older Kernel

1. At boot, press any key when the GRUB countdown appears.
2. Use the arrow keys to select the desired older kernel.
3. Press `Enter` to boot the selected kernel.

## Key Points

- Use `modprobe`, not `insmod` or `rmmod`, because it handles dependencies.
- New kernels install alongside existing kernels.
- GRUB displays installed kernels during boot, allowing selection of an older version when needed.
- RHEL commonly retains multiple kernel versions in `/boot`; verify the configured retention policy for the system.
- A tainted kernel can indicate that proprietary or unsupported modules have been loaded.
- After kernel updates, compare `uname -r` with `rpm -q kernel`.
- Many kernel-management commands read from `/proc` and present the information in a more usable format; prefer those commands over manually reading raw `/proc` files.