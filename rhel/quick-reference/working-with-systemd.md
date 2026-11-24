---
layout: default
title: systemd
parent: Quick Reference
grand_parent: RHEL
nav_order: 9
---

# systemd Units and Targets

## What Is a Unit?

A **unit** is any resource managed by `systemd`, such as services, sockets, timers, mounts, devices, and targets. Each unit is defined by a **unit file**, which describes the resource and how `systemd` should manage it.

## Managing Service State

```bash
# Start, stop, restart, or reload a unit
systemctl start <unit>
systemctl stop <unit>
systemctl restart <unit>
systemctl reload <unit>

# Show unit status (or full status output)
systemctl status <unit>
systemctl status -l <unit>

# Enable, enable immediately, or disable a unit at boot
systemctl enable <unit>
systemctl enable --now <unit>
systemctl disable <unit>

# Completely disable a unit (stronger than disable) or unmask it
systemctl mask <unit>
systemctl unmask <unit>

# Check whether a unit is enabled or active
systemctl is-enabled <unit>
systemctl is-active <unit>
```

### Common Unit Status Types

| Status | Meaning |
|---|---|
| `active (running)` | Service is running with one or more active processes. |
| `active (exited)` | One-time task completed successfully. |
| `inactive (dead)` | Service is not running. |
| `failed` | Service failed to start or terminated unexpectedly. |
| `enabled` | Unit is configured to start automatically at boot. |
| `disabled` | Unit is not configured to start automatically at boot. |

{: .note }
> `enabled` and `disabled` describe boot-time configuration. `active` and `inactive` describe the unit's current runtime state.

## Listing and Inspecting Units

```bash
# List all active units
systemctl list-units

# List active service units
systemctl --type=service

# List all service units, including inactive units
systemctl list-units --type=service --all

# List failed service units
systemctl --failed --type=service

# List available unit types
systemctl -t help

# Display unit file contents, resolved properties, or dependencies
systemctl cat <unit>
systemctl show <unit>
systemctl show -p MainPID --value <unit>
systemctl list-dependencies <unit>
```

### `systemctl cat` vs. `systemctl show`

| Command | Output | Best Used When |
|---|---|---|
| `systemctl cat <unit>` | Raw unit file plus any drop-in override files (with `# /path/to/file` headers) | Reading configured directives or checking which overrides exist on disk |
| `systemctl show <unit>` | All effective `Key=Value` properties (merged config, defaults, and runtime state) | Checking resolved values or querying a single property with `-p` |

## Modifying Unit Files

Use `systemctl edit` to create a drop-in override instead of modifying package-provided unit files directly.

| Location | Purpose |
|---|---|
| `/etc/systemd/system/` | Locally created unit files and custom overrides. |
| `/usr/lib/systemd/system/` | Package-provided default unit files; do not edit directly. |

```bash
# Create or edit a unit override
systemctl edit <unit>

# Reload systemd configuration after creating or editing unit files or overrides
systemctl daemon-reload
```

{: .important }
> Run `systemctl daemon-reload` after creating or editing unit files or drop-in overrides.

### Set the Default Editor

Add the following line to `/root/.bash_profile` and reload it:

```bash
export SYSTEMD_EDITOR="/bin/vi"
source /root/.bash_profile
```

## Working with Targets

A **target** is a `systemd` grouping mechanism that brings the system to a defined operational state by starting a set of related units (similar to SysV init runlevels).

| Target | Description |
|---|---|
| `poweroff.target` | Shut down the system. |
| `rescue.target` | Single-user recovery mode; root authentication is required. |
| `emergency.target` | Minimal emergency shell; root authentication is required. |
| `multi-user.target` | Multi-user mode without a GUI; typical server default. |
| `graphical.target` | Multi-user mode with a graphical user interface. |
| `reboot.target` | Reboot the system. |

```bash
# Show the default target
systemctl get-default

# Set default target for future boots
systemctl set-default multi-user.target
systemctl set-default graphical.target

# Switch to a target immediately (may stop services not required by the target)
systemctl isolate <target>

# List active targets
systemctl --type=target

# Show units started by a target
systemctl list-dependencies multi-user.target
```