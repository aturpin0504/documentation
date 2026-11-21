---
layout: default
title: Process Management
parent: Quick Reference
grand_parent: RHEL
nav_order: 6
---

# Process Management and Performance Tuning

## Process Types

- **Shell jobs:** Interactive commands run from a terminal.
- **Daemons:** Background services, such as `httpd` and `sshd`.
- **Kernel threads:** Processes with bracketed names, such as `[kthreadd]`; managed by the kernel and not directly user-controllable.

## Viewing Processes

```bash
# Show processes for the current user
ps

# Show all processes with CPU and memory usage (BSD style)
ps aux

# Show all processes with parent process IDs (POSIX style)
ps -ef

# Show process hierarchy in tree format
ps fax

# Filter processes
ps aux | grep httpd

# Find process IDs by name
pgrep dd
```

### `ps aux` vs. `ps -ef`

| Command | Style | Useful Information |
|---|---|---|
| `ps aux` | BSD | Resource usage, including `%CPU` and `%MEM`. |
| `ps -ef` | POSIX | Parent process ID (`PPID`). |

## Process States

| State | Code | Meaning |
|---|---|---|
| Running | `R` | Active or waiting for CPU time. |
| Sleeping | `S` | Waiting for an event. |
| Uninterruptible sleep | `D` | Waiting for I/O. |
| Stopped | `T` | Stopped, often with `Ctrl-Z`. |
| Zombie | `Z` | Process has exited but has not yet been reaped by its parent. |

View process states in the `STAT` column of `ps aux` (including modifiers such as `s`, `<`, and `+`) or the `S` column of `ps -ef` and `top`.

## Process Signals and Killing Processes

| Signal | Number | Purpose |
|---|---:|---|
| `SIGHUP` | 1 | Reload configuration. |
| `SIGKILL` | 9 | Force-stop a process; cannot be ignored or handled by the process. |
| `SIGTERM` | 15 | Graceful stop; default signal sent by `kill`. |

```bash
# Send SIGTERM (default)
kill 1234
kill -15 1234

# Force-kill a process
kill -9 1234

# Send SIGHUP to reload configuration
kill -1 1234

# List all signals
kill -l

# Kill all processes by exact name
killall dd

# Force-kill all processes by name
killall -9 dd

# Kill by name or pattern
pkill dd

# Kill all processes for a user
pkill -u username
```

{: .warning }
> `kill -9` sends `SIGKILL`, which can cause data loss and does not allow the process to perform cleanup actions. Always try `kill` or `kill -15` first.

## `top`: Interactive Process Monitor

```bash
top
```

### Common `top` Keys

| Key | Action |
|---|---|
| `k` | Kill a process |
| `r` | Renice a process |
| `M` | Sort by memory usage |
| `P` | Sort by CPU usage |
| `1` | Toggle individual CPU-core display |
| `h` | Show help |
| `q` | Quit |

## Load Average

```bash
# Show load average for 1, 5, and 15 minutes
uptime

# Show logged-in users, uptime, and load average
w

# Show CPU core count
lscpu | grep "^CPU(s)"
```

Load average represents the number of processes in runnable (`R`) or uninterruptible (`D`) state. As a rule of thumb, sustained load should not exceed the available CPU-core count (for example, a four-core system with a sustained load average of `8.0` is likely overloaded).

## Process Priority

### Nice Values

| Value | Meaning |
|---:|---|
| `-20` | Highest process priority |
| `0` | Default priority |
| `+19` | Lowest process priority |

Negative nice values require root privileges, and higher nice values mean **lower** scheduling priority.

```bash
# Start a command with lower priority
nice -n 10 command

# Start a command with higher priority (root required)
nice -n -10 command

# Change priority of a running process
renice -n 5 -p 1234

# Change priority by process name
renice -n 5 $(pgrep dd)
```

## `tuned`: Performance Tuning

```bash
# Show available profiles
tuned-adm list

# Show active profile
tuned-adm active

# Show recommended profile
tuned-adm recommend

# Set a profile (persists across reboots)
tuned-adm profile virtual-guest

# Enable and start tuned
systemctl enable --now tuned

# Stop tuned
systemctl stop tuned
```

### Common `tuned` Profiles

| Profile | Purpose |
|---|---|
| `balanced` | Default balance between power usage and performance. |
| `desktop` | Optimized for interactive desktop applications. |
| `throughput-performance` | Optimized for maximum throughput (common for servers). |
| `latency-performance` | Optimized for low-latency, time-sensitive applications. |
| `powersave` | Optimized for maximum power savings. |
| `virtual-guest` | Optimized for virtual machines. |
| `virtual-host` | Optimized for KVM hypervisors. |

Prefer `tuned` profiles over manual kernel-parameter tuning when an appropriate profile meets the operational requirement.