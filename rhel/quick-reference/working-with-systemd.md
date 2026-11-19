---
layout: default
title: systemd Units and Targets
parent: Quick Reference
grand_parent: RHEL
---

# systemd Units and Targets

## What Is a Unit?

A **unit** is any resource managed by `systemd`, such as:

- Services
- Sockets
- Timers
- Mounts
- Devices
- Targets

Each unit is defined by a **unit file**, which describes the resource and how `systemd` should manage it.

## Managing Service State

| Task | Command |
|---|---|
| Start a unit | `systemctl start <unit>` |
| Stop a unit | `systemctl stop <unit>` |
| Restart a unit | `systemctl restart <unit>` |
| Reload unit configuration | `systemctl reload <unit>` |
| Show unit status | `systemctl status <unit>` |
| Show detailed status | `systemctl status -l <unit>` |
| Enable unit at boot | `systemctl enable <unit>` |
| Enable and start immediately | `systemctl enable --now <unit>` |
| Disable unit from starting at boot | `systemctl disable <unit>` |
| Completely disable a unit | `systemctl mask <unit>` |
| Remove a unit mask | `systemctl unmask <unit>` |
| Check whether a unit is enabled | `systemctl is-enabled <unit>` |
| Check whether a unit is active | `systemctl is-active <unit>` |

## Common Unit Status Types

| Status | Meaning |
|---|---|
| `active (running)` | Service is running with one or more active processes. |
| `active (exited)` | One-time task completed successfully. |
| `inactive (dead)` | Service is not running. |
| `failed` | Service failed to start or terminated unexpectedly. |
| `enabled` | Unit is configured to start automatically at boot. |
| `disabled` | Unit is not configured to start automatically at boot. |

> [!NOTE]
> `enabled` and `disabled` describe boot-time configuration. `active` and `inactive` describe the unit's current runtime state.

## Listing Units

| Task | Command |
|---|---|
| List all active units | `systemctl list-units` |
| List active service units | `systemctl --type=service` |
| List all service units, including inactive units | `systemctl list-units --type=service --all` |
| List failed service units | `systemctl --failed --type=service` |
| List available unit types | `systemctl -t help` |

## Viewing Unit Information

| Task | Command |
|---|---|
| Display unit file contents | `systemctl cat <unit>` |
| Show all unit properties | `systemctl show <unit>` |
| Show unit dependencies | `systemctl list-dependencies <unit>` |

## Modifying Unit Files

Use `systemctl edit` to create a drop-in override instead of modifying package-provided unit files directly.

| Task | Command |
|---|---|
| Create or edit a unit override | `systemctl edit <unit>` |
| Reload `systemd` configuration | `systemctl daemon-reload` |

### Unit File Locations

| Location | Purpose |
|---|---|
| `/etc/systemd/system/` | Locally created unit files and custom overrides. |
| `/usr/lib/systemd/system/` | Package-provided default unit files; do not edit directly. |

> [!IMPORTANT]
> Run `systemctl daemon-reload` after creating or editing unit files or drop-in overrides.

### Set the Default Editor

Add the following line to `/root/.bash_profile`:

```bash
export SYSTEMD_EDITOR="/bin/vi"
```

Reload the profile or open a new shell:

```bash
source /root/.bash_profile
```

## What Is a Target?

A **target** is a `systemd` grouping mechanism that brings the system to a defined operational state by starting a set of related units.

Targets are similar to SysV init runlevels. For example, `multi-user.target` starts the services needed for a multi-user, non-graphical system.

## Working with Targets

| Task | Command |
|---|---|
| Show the default target | `systemctl get-default` |
| Set default target to multi-user mode | `systemctl set-default multi-user.target` |
| Set default target to graphical mode | `systemctl set-default graphical.target` |
| Switch to a target immediately | `systemctl isolate <target>` |
| List active targets | `systemctl --type=target` |
| Show units started by a target | `systemctl list-dependencies multi-user.target` |

## Common Targets

| Target | Description |
|---|---|
| `poweroff.target` | Shut down the system. |
| `rescue.target` | Single-user recovery mode; root login is required. |
| `emergency.target` | Minimal emergency shell; root login is required. |
| `multi-user.target` | Multi-user mode without a GUI; typical server default. |
| `graphical.target` | Multi-user mode with a graphical user interface. |
| `reboot.target` | Reboot the system. |

## Key Points

- `systemctl set-default` changes the target used at the next boot.
- `systemctl isolate` switches to a target immediately.
- `rescue.target` and `emergency.target` both require root authentication.
- `systemctl isolate` can stop services not required by the target and place the system into a recovery-oriented state.
- `systemctl mask` prevents a unit from being started, making it stronger than `systemctl disable`.
- `multi-user.target` is the typical target for servers.
- Use `systemctl list-dependencies <target>` to identify units associated with a target.