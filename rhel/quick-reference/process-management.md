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
# Show processes for the current user and terminal
ps

# Show all processes with CPU and memory usage (BSD style)
ps aux

# Show all processes with parent process IDs (POSIX style)
ps -ef

# Show process hierarchy in tree format
ps fax

# Filter processes by text match
ps aux | grep httpd

# Find process IDs by name
pgrep dd

# Show PID and process name for all processes
ps -eo pid,comm

# Preview pgrep matches
pgrep -l httpd
pgrep -lx dd
pgrep -af "app.py"
```

> Prefer `pgrep` over `ps aux | grep` when possible. The `grep` command can appear in its own search results.

### `ps aux` vs. `ps -ef`

| Command | Style | Useful Information |
|---|---|---|
| `ps aux` | BSD | Resource usage, including `%CPU`, `%MEM`, virtual memory size (`VSZ`), and resident memory (`RSS`). |
| `ps -ef` | POSIX | Parent process ID (`PPID`), process start time, and full command line. |

### `pgrep` / `pkill` Matching Options

| Option | Purpose |
|---|---|
| `-l` | List PID and process name (`comm`) |
| `-x` | Match the exact process name only |
| `-f` | Match against the full command line instead of `comm` |
| `-a` | Print the full command line; commonly combined as `-af` |

{: .note }
> `pgrep` and `pkill` perform substring matching on the process name by default. For example, `pgrep dd` can also match `sshd` and `httpd`. Preview matches with `pgrep -l` or `pgrep -af` before running `pkill`.

## Process States

| State | Code | Meaning |
|---|---|---|
| Running | `R` | Running or waiting for CPU time. |
| Interruptible sleep | `S` | Waiting for an event. |
| Uninterruptible sleep | `D` | Usually waiting for I/O. |
| Stopped | `T` | Stopped, often with `Ctrl-Z` or a signal. |
| Zombie | `Z` | Process has exited but has not yet been reaped by its parent. |

View process states in:

- The `STAT` column of `ps aux`
- The `S` column of `ps -ef`
- The process list in `top`

The `STAT` column can also contain modifiers such as:

| Modifier | Meaning |
|---|---|
| `<` | High-priority process |
| `N` | Low-priority process |
| `s` | Session leader |
| `l` | Multi-threaded process |
| `+` | Process is in the foreground process group |

## Process Signals and Killing Processes

| Signal | Number | Purpose |
|---|---:|---|
| `SIGHUP` | 1 | Traditionally requests a configuration reload or terminal hangup handling. |
| `SIGKILL` | 9 | Force-stops a process; cannot be ignored or handled by the process. |
| `SIGTERM` | 15 | Requests a graceful stop; default signal sent by `kill`. |

```bash
# Send SIGTERM (default)
kill 1234
kill -15 1234

# Force-kill a process
kill -9 1234

# Send SIGHUP to request a configuration reload
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
> `kill -9` sends `SIGKILL`, which can cause data loss and does not allow the process to close files, flush data, or perform cleanup actions. Always try `kill` or `kill -15` first.

## `top`: Interactive Process Monitor

```bash
top
```

`top` provides a real-time view of processes, CPU activity, memory use, load average, and system uptime.

### Common `top` Keys

| Key | Action |
|---|---|
| `k` | Send a signal to a process |
| `r` | Renice a process |
| `M` | Sort by memory usage |
| `P` | Sort by CPU usage |
| `1` | Toggle individual logical CPU statistics |
| `H` | Toggle individual threads |
| `m` | Cycle memory-summary display modes |
| `f` | Select or remove displayed fields |
| `h` | Show help |
| `q` | Quit |

## Memory Monitoring

Use memory commands to determine installed RAM, currently available memory, swap use, and the processes consuming the most resident memory.

```bash
# Show total, used, available, and swap memory in human-readable units
free -h

# Refresh memory statistics every second
free -h -s 1

# Display detailed kernel memory statistics
cat /proc/meminfo

# Display selected memory and swap values
grep -E 'MemTotal|MemAvailable|SwapTotal|SwapFree' /proc/meminfo

# Display memory, paging, I/O, and CPU statistics every second
vmstat 1

# Show processes using the most resident memory
ps -eo pid,user,comm,%mem,rss --sort=-rss | head -n 11

# Show processes using the most virtual memory
ps -eo pid,user,comm,vsz,rss --sort=-vsz | head -n 11
```

### Memory Fields

| Field | Meaning |
|---|---|
| `MemTotal` | Total installed physical memory. |
| `MemAvailable` | Estimated memory available for new workloads without significant swapping. |
| `SwapTotal` | Total configured swap space. |
| `SwapFree` | Unused swap space. |
| `%MEM` | Percentage of physical memory used by a process. |
| `RSS` or `RES` | Resident Set Size: physical memory currently used by a process and not swapped out. |
| `VSZ` or `VIRT` | Virtual address space assigned to a process; not the same as physical memory consumption. |

> **Interpretation:** Low `free` memory alone is not necessarily a problem. Linux uses otherwise idle RAM for filesystem cache. Focus on `available` memory, swap activity, sustained paging, and application behavior when evaluating memory pressure.

### Memory Usage in `top`

```bash
top
```

| Key | Action |
|---|---|
| `M` | Sort processes by memory usage. |
| `m` | Cycle memory-summary display modes. |
| `f` | Select or remove displayed fields. |
| `q` | Quit. |

## Load Average

```bash
# Show logged-in users, uptime, and load average
uptime

# Show logged-in users, uptime, load average, and active processes
w

# Show the number of processing units available to the current environment
nproc

# Show detailed CPU architecture and topology information
lscpu
```

Load average represents the average number of processes in runnable (`R`) or uninterruptible (`D`) state over the previous 1, 5, and 15 minutes.

As a general rule of thumb, sustained load significantly higher than the number of available CPU processing units can indicate CPU contention, I/O waits, or another system bottleneck.

For example, a system with four available CPUs and a sustained load average of `8.0` warrants investigation.

## CPU Monitoring

Use CPU commands to identify available processors, overall CPU activity, and processes consuming the most CPU time.

```bash
# Show detailed CPU architecture and topology information
lscpu

# Show the number of processing units available to the current environment
nproc

# Show processes using the most CPU
ps -eo pid,user,comm,%cpu,time --sort=-%cpu | head -n 11

# Display process and overall CPU activity interactively
top

# Display process, memory, paging, I/O, and CPU activity every second
vmstat 1

# Display load average for the previous 1, 5, and 15 minutes
uptime
```

If the `sysstat` package is installed, use `mpstat` for CPU utilization statistics:

```bash
# Display CPU utilization summary every second
mpstat 1

# Display utilization for each logical CPU every second
mpstat -P ALL 1
```

### CPU Fields

| Field | Meaning |
|---|---|
| `%CPU` | CPU use attributed to a process. A multi-threaded process can exceed `100%` on a multi-core system. |
| `TIME` or `TIME+` | Cumulative CPU time consumed by the process. |
| `us` | CPU time spent running user-space processes. |
| `sy` | CPU time spent running kernel code. |
| `id` | Idle CPU time. |
| `wa` | CPU time spent waiting for I/O operations. |
| `st` | CPU time taken by the hypervisor on a virtual machine. |

### CPU Usage in `top`

```bash
top
```

| Key | Action |
|---|---|
| `P` | Sort processes by CPU usage. |
| `1` | Toggle individual logical CPU statistics. |
| `H` | Toggle individual threads. |
| `f` | Select or remove displayed fields. |
| `q` | Quit. |

## Process Priority

### Nice Values

| Value | Meaning |
|---:|---|
| `-20` | Highest scheduling priority |
| `0` | Default priority |
| `+19` | Lowest scheduling priority |

Negative nice values generally require root privileges or the appropriate capability. Higher nice values mean **lower** scheduling priority.

```bash
# Start a command with lower priority
nice -n 10 command

# Start a command with higher priority
# Requires root privileges or appropriate capability
nice -n -10 command

# Change priority of a running process
renice -n 5 -p 1234

# Change priority for processes matching a name
renice -n 5 -p $(pgrep -x dd)
```

{: .warning }
> Verify `pgrep` results before using command substitution with `renice`, `kill`, or another process-management command. Use `pgrep -lx dd` to preview exact-name matches.