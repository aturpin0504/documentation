---
layout: default
title: Kernel Management
parent: Quick Reference
grand_parent: RHEL
nav_order: 20
---

# Kernel Information and Management

## Kernel Overview

The Linux kernel is the core of the operating system. It manages CPU scheduling, memory, storage, networking, device input/output, and communication between software and hardware.

The kernel does not normally translate every application instruction into CPU instructions. Instead, applications request operating-system services through system calls, and the kernel coordinates access to CPU time, memory, drivers, and hardware resources.

## Kernel and System Information

```bash
# Show the running kernel release
uname -r

# Show kernel, hostname, architecture, and other system information
uname -a

# Show detailed host information, including virtualization
hostnamectl status

# Show the installed RHEL release
cat /etc/redhat-release

# Show the operating system and kernel release information
cat /etc/os-release
uname -r
```

### Memory Aids

- `uname` — Unix name; displays kernel and system identity information.
- `dmesg` — Display messages; shows the kernel ring buffer.
- `udev` — Userspace device manager; processes hardware events and applies device rules.
- `/proc` — Virtual filesystem containing runtime kernel and process information.
- `/sys` — Virtual filesystem exposing devices, drivers, buses, and kernel objects.

## Kernel Threads

Kernel threads are tasks created by the kernel to perform internal work, such as memory management, device handling, writeback, or work queues.

When displayed by commands such as `ps aux`, kernel threads commonly have names in square brackets:

```text
[kthreadd]
[kworker/0:1]
[kswapd0]
```

```bash
# Display all processes and kernel threads
ps aux

# Show tasks with bracketed names
ps -e -o pid,ppid,stat,comm | grep '^\|'
```

Common examples:

| Kernel Thread | Purpose |
|---|---|
| `[kthreadd]` | Parent task for many kernel threads. |
| `[kworker/*]` | Kernel worker threads that process deferred work. |
| `[kswapd0]` | Kernel memory-reclamation thread. |
| `[ksoftirqd/*]` | Processes deferred software interrupt work. |

{: .note }
> Kernel threads are not normal user-space processes. Do not attempt to stop, kill, or renice them as a troubleshooting action.

## Kernel Messages

`dmesg` displays messages from the kernel ring buffer. Kernel messages are useful for investigating hardware detection, driver loading, storage errors, network-link changes, kernel warnings, and boot problems.

```bash
# View recent kernel messages
dmesg

# View kernel messages with a pager
dmesg | less

# Display human-readable timestamps
dmesg -T | less

# Display kernel messages from the system journal
journalctl -k

# Display kernel messages from the current boot
journalctl -k -b

# Display kernel errors from the current boot
journalctl -k -b -p err

# Search for storage, network, or driver messages
journalctl -k -b | grep -Ei 'error|fail|warn|driver|firmware'
```

### Understanding `dmesg` Timestamps

A normal `dmesg` timestamp is relative to kernel startup:

```text
[   12.345678] device initialization message
```

The value in brackets means the message was recorded approximately `12.345678` seconds after the kernel started.

This helps determine how long after boot an issue occurred:

- A timestamp near `0` usually indicates a boot-time event.
- A timestamp of `[3600.0]` occurred about one hour after boot.
- Use `dmesg -T` for a human-readable approximation of the timestamp.

{: .note }
> Human-readable `dmesg -T` timestamps can be inaccurate after suspend and resume events. Use `journalctl -k` when you need wall-clock timestamps and persistent kernel logs.

## Kernel Modules

The kernel is modular. Core functionality is built into the kernel image, while optional functionality can be loaded as kernel modules.

Kernel modules can provide:

- Hardware drivers, such as network, storage, USB, graphics, and sound drivers.
- Filesystem support, such as NFS, XFS, or other optional filesystem features.
- Network protocols, filtering, and security features.
- Virtualization, device-mapper, and storage functionality.
- Other optional kernel capabilities.

RHEL normally loads the required modules automatically during boot or when hardware is detected. Open-source, vendor-supported drivers included with RHEL are preferred because they receive normal RHEL testing, updates, and support.

Third-party, out-of-tree, or proprietary kernel modules can complicate troubleshooting and may taint the running kernel.

```bash
# List loaded modules
lsmod

# Check whether a specific module is loaded
lsmod | grep -w module_name

# Example: check whether the virtio_net driver is loaded
lsmod | grep -w virtio_net

# Display the module directory for the running kernel
ls /lib/modules/"$(uname -r)"

# Display modules matching a name
find /lib/modules/"$(uname -r)" -type f -name '*module_name*'
```

### Understanding `lsmod`

`lsmod` displays currently loaded kernel modules.

Example output:

```text
Module                  Size  Used by
virtio_net             73728  0
virtio_blk             20480  0
```

| Column | Meaning |
|---|---|
| `Module` | Name of the loaded kernel module. |
| `Size` | Approximate memory size of the loaded module. |
| `Used by` | Reference count or dependent-module information. |

```bash
# Show the column header and first few loaded modules
lsmod | head

# Show only the first 20 lines
lsmod | head -n 20

# Search loaded modules for a specific module name
lsmod | grep -w xfs
```

## Working with Kernel Modules

### Module Commands

| Command | Purpose |
|---|---|
| `lsmod` | Lists modules currently loaded into the running kernel. |
| `modinfo <module>` | Displays module metadata, dependencies, aliases, license, filename, and supported parameters. |
| `modprobe <module>` | Loads a module and any required dependencies. |
| `modprobe -r <module>` | Removes a module and, when possible, unused dependent modules. |
| `modprobe -n -v <module>` | Shows what `modprobe` would do without changing the running kernel. |

```bash
# Display information about a module
modinfo module_name

# Display the module filename only
modinfo -n module_name

# Display the module license
modinfo -F license module_name

# Display module dependencies
modinfo -F depends module_name

# Display module aliases
modinfo -F alias module_name

# Display supported module parameters
modinfo -p module_name

# Preview a module load without making changes
modprobe -n -v module_name

# Load a module
modprobe module_name

# Load a module with a temporary parameter
modprobe cdrom debug=1

# Remove a module
modprobe -r module_name
```

{: .warning }
> Do not remove a module that is actively supporting a storage device, network interface, mounted filesystem, virtual machine, or other required service. `modprobe -r` can fail when the module is in use, which is usually the desired safety behavior.

Use module names without a `.ko`, `.ko.xz`, or other file extension:

```bash
# Correct
modprobe virtio_net

# Incorrect
modprobe virtio_net.ko.xz
```

### Persistent Module Parameters

Module parameters modify the behavior of a specific kernel module. They are usually needed only for hardware compatibility, troubleshooting, feature enablement, performance tuning, or a vendor-supported configuration requirement.

Most systems do not require custom module parameters.

```bash
# View parameters supported by a module
modinfo -p module_name

# View values for parameters of a currently loaded module
ls /sys/module/module_name/parameters/

# Read a current parameter value
cat /sys/module/module_name/parameters/parameter_name
```

Example:

```bash
# View parameters supported by the cdrom module
modinfo -p cdrom

# View current values if the module is loaded
ls /sys/module/cdrom/parameters/
```

To make a module parameter persistent, create a configuration file under `/etc/modprobe.d/`:

```ini
# /etc/modprobe.d/cdrom.conf
options cdrom debug=1
```

After adding a persistent module configuration, reboot or unload and reload the module when it is safe to do so.

```bash
# Review active modprobe configuration
modprobe -c | grep -E '^(options|blacklist).*module_name'

# Reload a module only when it is safe and not in active use
modprobe -r module_name
modprobe module_name
```

## PCI Devices and Drivers

`lspci` displays devices attached to the PCI or PCI Express bus. It is useful for investigating network adapters, storage controllers, graphics adapters, RAID controllers, and other PCI devices.

```bash
# List PCI devices
lspci

# List PCI devices with numeric vendor and device IDs
lspci -nn

# List PCI devices and associated kernel drivers/modules
lspci -k

# Show the first several PCI devices
lspci | head

# Show the first several PCI devices with driver information
lspci -k | head -n 30

# Search for Ethernet adapters
lspci -nnk | grep -A 3 -i ethernet

# Search for storage controllers
lspci -nnk | grep -A 3 -Ei 'storage|sata|scsi|raid|nvme'

# Inspect one PCI device by bus address
lspci -k -s 03:00.0
```

### `lspci -k`

The `-k` option displays:

- The kernel driver currently in use by the PCI device.
- Kernel modules that can support the device.

Example output:

```text
03:00.0 Ethernet controller: Example Device
        Subsystem: Example Device
        Kernel driver in use: example_driver
        Kernel modules: example_driver
```

| Field | Meaning |
|---|---|
| `Kernel driver in use` | The driver currently bound to the device. |
| `Kernel modules` | Modules that may support the device. |
| No driver in use | The device might be unsupported, disabled, unbound, missing firmware, or experiencing a driver-load failure. |

### Troubleshoot a Device Without a Loaded Driver

Use this workflow when hardware is visible but does not appear to function.

```bash
# 1. Identify the PCI device and check its driver
lspci -nnk

# 2. Inspect one device in detail
lspci -k -s <bus-address>

# 3. Check whether the expected module is loaded
lsmod | grep -w <module-name>

# 4. Inspect available module information
modinfo <module-name>

# 5. Preview the module load
modprobe -n -v <module-name>

# 6. Load the module, if appropriate and authorized
modprobe <module-name>

# 7. Review kernel messages for driver or firmware errors
journalctl -k -b | grep -Ei '<module-name>|firmware|error|fail'
```

If `lspci -k` shows a device but no `Kernel driver in use` line, investigate:

1. Whether the correct driver module exists for the running kernel.
2. Whether the module loaded successfully.
3. Whether required firmware is installed.
4. Whether the device is disabled in firmware or virtual-machine configuration.
5. Whether the hardware is supported by the installed RHEL version and kernel.
6. Whether an unsupported third-party driver or kernel module is required.

For USB devices, use `lsusb` instead of `lspci`:

```bash
# List USB devices
lsusb

# Display the USB device tree and drivers
lsusb -t

# Monitor kernel messages while connecting a USB device
journalctl -k -f
```

## `systemd-udevd` and Device Events

`systemd-udevd` is the userspace device manager. It receives device events from the kernel when hardware is added, removed, or changes state.

It processes udev rules to:

- Create and manage device nodes under `/dev`.
- Set device ownership and permissions.
- Create stable device symlinks, such as `/dev/disk/by-id/`.
- Apply device-specific configuration.
- Trigger approved actions associated with matching udev rules.

```bash
# Check udev service status
systemctl status systemd-udevd

# Display information about a device node
udevadm info --query=all --name=/dev/sda

# Display device properties
udevadm info --query=property --name=/dev/sda

# Test which rules apply to a device path
udevadm test /sys/class/block/sda
```

### Using `udevadm monitor`

You do not need `udevadm monitor` for routine administration. Use it when troubleshooting device detection, hot-plug behavior, USB devices, storage devices, NICs, or custom udev rules.

```bash
# Monitor both kernel and udev events
udevadm monitor --kernel --udev

# Monitor events and display device properties
udevadm monitor --kernel --udev --property

# Monitor only block-device events
udevadm monitor --kernel --udev \
  --subsystem-match=block
```

A useful troubleshooting workflow:

1. Start `udevadm monitor --kernel --udev --property`.
2. Plug in or remove the device, or trigger the suspected device event.
3. Observe the kernel event.
4. Observe the udev event after rule processing.
5. Review `journalctl -k -f` for driver, firmware, or hardware errors.

## `/proc` and `/sys`

You normally use standard commands instead of reading raw `/proc` files. Commands such as `ps`, `free`, `uptime`, `lsmod`, `sysctl`, `lscpu`, and `ip` present information from `/proc` or `/sys` in a more usable form.

Use `/proc` and `/sys` when troubleshooting, verifying a specific kernel value, or working with an administrative procedure that explicitly references those files.

| Location | Purpose | Common Command Alternative |
|---|---|---|
| `/proc/cpuinfo` | CPU details | `lscpu` |
| `/proc/meminfo` | Memory details | `free -h` |
| `/proc/modules` | Loaded kernel modules | `lsmod` |
| `/proc/cmdline` | Kernel boot parameters | `cat /proc/cmdline` |
| `/proc/sys/` | Runtime kernel tunables | `sysctl` |
| `/proc/sys/kernel/tainted` | Kernel taint status | `cat /proc/sys/kernel/tainted` |
| `/sys/module/` | Loaded-module information and parameters | `modinfo`, `lsmod`, and `/sys/module/<module>/parameters/` |
| `/sys/class/` | Device classes, such as block and network devices | `ip link`, `lsblk`, and `udevadm info` |

```bash
# Display kernel boot parameters
cat /proc/cmdline

# Display kernel taint value
cat /proc/sys/kernel/tainted

# Display a kernel tunable
sysctl kernel.hostname

# Display all kernel tunables
sysctl -a | less

# Display memory information directly
cat /proc/meminfo

# Use the normal command instead
free -h
```

{: .note }
> Prefer normal administrative commands for routine work. Use `/proc` and `/sys` when you need lower-level details, a current kernel value, or a troubleshooting data source not exposed by another command.

## Kernel Taint Status

A kernel can become tainted when it loads unsupported, proprietary, externally built, force-loaded, or otherwise nonstandard components.

```bash
# Search kernel messages for taint indications
dmesg | grep -i taint

# Check the kernel taint bitmask
cat /proc/sys/kernel/tainted

# View loaded modules and inspect third-party modules
lsmod
modinfo <module-name>
```

A taint value of `0` indicates no taint flags are set.

{: .warning }
> A tainted kernel is not automatically broken, but it can complicate troubleshooting and vendor support. Prefer RHEL-provided or vendor-supported kernel modules. Review organizational policy before installing proprietary or out-of-tree drivers.

## Kernel Updates and Reboot Checks

Kernel updates install alongside existing kernels, allowing rollback to an earlier kernel if necessary.

```bash
# Update the kernel and required dependencies
dnf update kernel

# List installed kernel meta packages
rpm -q kernel

# List installed kernel-core packages
rpm -q kernel-core

# Display the kernel configured as the default boot entry
grubby --default-kernel

# Check whether a reboot is required, if needs-restarting is installed
needs-restarting -r
```

### Check Whether a Reboot Is Required

Compare the running kernel to the kernel configured as the default boot entry:

```bash
# Show running kernel version
uname -r

# Show default boot kernel path
grubby --default-kernel
```

A default boot path such as:

```text
/boot/vmlinuz-5.14.0-<release>.x86_64
```

corresponds to the kernel release:

```text
5.14.0-<release>.x86_64
```

If the running kernel does not match the default boot kernel, schedule or perform a reboot according to the approved maintenance procedure.

### After a Kernel Update

```bash
# Reboot to load the configured default kernel
reboot

# Verify the running kernel after reboot
uname -r

# Verify the default boot entry
grubby --default-kernel

# Check whether a reboot is still required
needs-restarting -r
```

{: .note }
> On a Satellite server, install or update packages through the approved `satellite-maintain` workflow. This page's kernel verification and reboot checks remain useful, but do not use direct `dnf` package operations on Satellite unless specifically authorized by the applicable Satellite maintenance procedure.

## Automatic Reboot Check After a Kernel Update

The following script compares the currently running kernel to the kernel configured as the default GRUB boot entry. If they differ, it schedules a reboot in one minute.

This is more reliable than selecting the newest kernel only by RPM installation time because it checks the kernel that the system is actually configured to boot.

```bash
#!/usr/bin/env bash

RUNNING_KERNEL=$(uname -r)
DEFAULT_KERNEL=$(grubby --default-kernel)
DEFAULT_KERNEL=${DEFAULT_KERNEL##*/vmlinuz-}

if [[ "$RUNNING_KERNEL" != "$DEFAULT_KERNEL" ]]; then
    echo "Kernel update detected."
    echo "Running kernel: $RUNNING_KERNEL"
    echo "Default boot kernel: $DEFAULT_KERNEL"
    echo "Scheduling reboot in 1 minute..."

    shutdown -r +1 \
      "Scheduled reboot: kernel changed from $RUNNING_KERNEL to $DEFAULT_KERNEL"
else
    echo "Running kernel matches the default boot kernel. No reboot required."
fi
```

### Script Breakdown

| Line or Command | Purpose |
|---|---|
| `uname -r` | Retrieves the currently running kernel release. |
| `grubby --default-kernel` | Displays the kernel image configured for the next normal boot. |
| `${DEFAULT_KERNEL##*/vmlinuz-}` | Removes the `/boot/vmlinuz-` path prefix. |
| `[[ "$RUNNING_KERNEL" != "$DEFAULT_KERNEL" ]]` | Tests whether the current kernel differs from the default boot kernel. |
| `shutdown -r +1` | Schedules a reboot one minute from now. |
| `else` block | Confirms that no reboot is required when versions match. |

{: .warning }
> Run a reboot-scheduling script only through an approved maintenance workflow. A scheduled reboot can interrupt active users, package transactions, backups, database activity, or other critical services.

## Reinstalling a Kernel Package

Use `dnf reinstall` first when a kernel package appears damaged or incompletely installed.

Before reinstalling or removing a kernel version:

1. Confirm the exact affected kernel version.
2. Confirm that at least one known-good alternate kernel remains installed.
3. Do not remove the kernel currently running unless following a specific recovery procedure.
4. Ensure the required repositories are available.
5. Review the transaction before accepting it.

```bash
# Show the running kernel
uname -r

# List installed kernel packages
rpm -qa | grep -E '^kernel(|-core|-modules|-modules-core|-modules-extra)-'

# List all packages associated with a specific kernel release
rpm -qa | grep '5.14.0-<release>'
```

### Normal Kernel Reinstall

Reinstall the affected kernel packages without removing them first:

```bash
# Reinstall the kernel meta package and primary subpackages
dnf reinstall -y \
  kernel-5.14.0-<release>.x86_64 \
  kernel-core-5.14.0-<release>.x86_64 \
  kernel-modules-5.14.0-<release>.x86_64 \
  kernel-modules-core-5.14.0-<release>.x86_64
```

If installed for that release, include optional associated packages as applicable:

```bash
# Example: reinstall extra modules when they are installed
dnf reinstall -y \
  kernel-modules-extra-5.14.0-<release>.x86_64
```

After reinstalling:

```bash
# Verify installed kernel packages
rpm -qa | grep '5.14.0-<release>'

# Rebuild module dependency metadata if directed by troubleshooting guidance
depmod -a

# Reboot into the repaired kernel when approved
reboot
```

### Remove and Reinstall a Non-Running Kernel

Use removal only as a last resort when a normal reinstall fails and the affected kernel is **not** the currently running kernel.

```bash
# Confirm the currently running kernel before proceeding
uname -r

# Remove the affected non-running kernel packages
dnf remove -y \
  kernel-5.14.0-<release>.x86_64 \
  kernel-core-5.14.0-<release>.x86_64 \
  kernel-modules-5.14.0-<release>.x86_64 \
  kernel-modules-core-5.14.0-<release>.x86_64

# Reinstall the specified kernel version
dnf install -y \
  kernel-5.14.0-<release>.x86_64
```

{: .warning }
> Do not remove the currently running kernel or the only remaining bootable kernel. When removing a kernel version, review all associated `kernel-core`, `kernel-modules`, `kernel-modules-core`, and optional `kernel-modules-extra` packages for that version.

## Booting an Older Kernel

1. Reboot the system.
2. Press a key when the GRUB countdown appears.
3. Select **Advanced options for Red Hat Enterprise Linux**.
4. Select the desired earlier kernel version.
5. Press `Enter` to boot the selected kernel.
6. Verify the running kernel after login.

```bash
# Verify the currently running kernel
uname -r

# List installed kernel versions
rpm -q kernel
rpm -q kernel-core

# Review all GRUB kernel entries
grubby --info=ALL | grep -E '^kernel=|^title='
```

## Key Points

- Keep the RHEL-provided kernel updated through approved package-management processes.
- RHEL commonly retains multiple installed kernels to support rollback.
- Most systems load required drivers automatically; manually managing modules is usually limited to troubleshooting, hardware support, module parameters, or approved vendor requirements.
- Use `lsmod | grep -w <module>` to check whether a module is loaded.
- Use `modinfo <module>` to inspect a module before attempting to load it.
- Use `lspci -k` to identify PCI devices and determine which kernel driver is bound to them.
- Use `journalctl -k -b` and `dmesg` to troubleshoot driver, firmware, and hardware errors.
- Use `udevadm monitor` only when diagnosing device-add, device-remove, hot-plug, or udev-rule behavior.
- Prefer standard commands over raw `/proc` files for routine administration.
- Treat proprietary, third-party, and out-of-tree kernel modules as a support and change-management concern.
