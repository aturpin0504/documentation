---
layout: default
title: Cron & At
parent: Quick Reference
grand_parent: RHEL
nav_order: 10
---

# Cron and At Service Management

## Cron Service Overview

`cron` is a service for running processes automatically at scheduled times. It consists of the `crond` daemon and cron configuration files.

```bash
# Check the current status of the cron service
systemctl status crond

# Show current service state with full output
systemctl status crond -l
```

{: .note }
> `crond` wakes up every minute to check its configuration. Configuration changes do not normally require a service reload or restart.

## Cron Configuration Locations

| Location | Description |
|---|---|
| `/etc/crontab` | Main system cron configuration file. View for reference; do not modify directly. |
| `/etc/cron.d/` | Cron files for services, often installed by RPM packages. |
| `/etc/cron.hourly/` | Scripts executed hourly. |
| `/etc/cron.daily/` | Scripts executed daily. |
| `/etc/cron.weekly/` | Scripts executed weekly. |
| `/etc/cron.monthly/` | Scripts executed monthly. |
| `/var/spool/cron/` | User-specific cron files. Do not edit directly. Use `crontab -e`. |

## Managing User Cron Jobs

```bash
# Edit or list the current user's cron jobs
crontab -e
crontab -l

# Edit or list a specific user's cron jobs (run as root)
crontab -e -u username
crontab -l -u username

# Remove the current user's or a specific user's crontab
crontab -r
crontab -r -u username
```

{: .warning }
> `crontab -r` deletes the crontab without opening an editor. Verify the intended user before running it.

## Cron Job Format

### User Crontab Format

```text
minute hour day-of-month month day-of-week command
```

| Field | Valid Range |
|---|---|
| Minute | `0-59` |
| Hour | `0-23` |
| Day of month | `1-31` |
| Month | `1-12` |
| Day of week | `0-7`; Sunday is `0` or `7` |

Example user crontab entry (runs at 2:00 AM, Monday through Friday):

```cron
0 2 * * 1-5 logger message from root
```

### `/etc/cron.d/` Format

Files in `/etc/cron.d/` include an additional **username** field:

```text
minute hour day-of-month month day-of-week username command
```

Example (runs at minute 11 of every hour as `root`):

```cron
11 * * * * root logger message from /etc/cron.d
```

## Environment Variables in Crontab

| Variable | Purpose |
|---|---|
| `SHELL` | Shell used to run cron commands; default is typically `/bin/bash`. |
| `PATH` | Search path used to find commands. |
| `MAILTO` | Email recipient for cron job output. |

Set variables at the top of the crontab file:

```cron
SHELL=/bin/bash
PATH=/usr/local/bin:/usr/bin:/bin
MAILTO=admin@example.com

0 2 * * * /path/to/script.sh
```

## Cron Access Control

| File | Purpose |
|---|---|
| `/etc/cron.allow` | If present, users must be listed in this file to use `cron`. |
| `/etc/cron.deny` | If present, users listed in this file are denied use of `cron`. |

{: .note }
> `/etc/cron.allow` and `/etc/cron.deny` should not both exist simultaneously. If neither file exists, only `root` can use `cron`.

## Cron Operational Checks

- Allow a three-minute margin between saving a cron configuration and the intended execution time.
- Scripts in `/etc/cron.hourly/`, `/etc/cron.daily/`, `/etc/cron.weekly/`, and `/etc/cron.monthly/` require execute permissions.
- No single command displays every scheduled cron job; check user crontabs (`crontab -l`), `/etc/crontab`, `/etc/cron.d/`, and `/etc/cron.{hourly,daily,weekly,monthly}/`.

```bash
# Ensure drop-in cron scripts are executable
chmod +x /etc/cron.daily/example-script

# Review /var/log/messages for cron activity
grep cron /var/log/messages
grep CROND /var/log/messages
```

## At Service: One-Time Task Scheduling

The `at` service schedules commands to run once at a specified time. Ensure the `atd` service is running and scripts are executable before scheduling.

```bash
# Check the atd service status
systemctl status atd

# Schedule one-time jobs
at 15:00
at teatime
at noon
at tomorrow

# View or remove scheduled at jobs
atq
atrm jobnumber

# Execute a job when system load permits (typically below 0.8)
batch
```

### Interactive `at` Session

1. Start an `at` session with `at 15:00`.
2. Enter the command or script path (e.g., `/path/to/script.sh`).
3. Press `Ctrl-D` to close the `at` prompt and save the job.
4. Verify with `atq`.

```text
$ at 15:00
at> logger "One-time maintenance task started"
at> <Ctrl-D>
```