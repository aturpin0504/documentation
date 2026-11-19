---
layout: default
title: Cron and At Service Management
parent: Quick Reference
grand_parent: RHEL
---

# Cron and At Service Management

## Cron Service Overview

`cron` is a service for running processes automatically at scheduled times. It consists of the `crond` daemon and cron configuration files.

## Managing the `crond` Service

| Command | Description |
|---|---|
| `systemctl status crond` | Check the current status of the cron service. |
| `systemctl status crond -l` | Show current service state with full output. |

> [!NOTE]
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

| Command | Description |
|---|---|
| `crontab -e` | Edit the current user's cron jobs. |
| `crontab -e -u username` | Edit a specific user's cron jobs; run as root. |
| `crontab -l` | List the current user's cron jobs. |
| `crontab -l -u username` | List a specific user's cron jobs. |
| `crontab -r` | Remove the current user's crontab. |
| `crontab -r -u username` | Remove a specific user's crontab; run as root. |

> [!WARNING]
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

Example user crontab entry:

```cron
0 2 * * 1-5 logger message from root
```

This runs at 2:00 AM, Monday through Friday.

### `/etc/cron.d/` Format

Files in `/etc/cron.d/` include an additional **username** field:

```text
minute hour day-of-month month day-of-week username command
```

Example:

```cron
11 * * * * root logger message from /etc/cron.d
```

This runs at minute 11 of every hour as `root`.

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

> [!NOTE]
> `/etc/cron.allow` and `/etc/cron.deny` should not both exist simultaneously. If neither file exists, only `root` can use `cron`.

## Cron Operational Tips

- Allow a three-minute margin between saving a cron configuration and the intended execution time.
- Verify that the `crond` service is running.
- Scripts in `/etc/cron.hourly/`, `/etc/cron.daily/`, `/etc/cron.weekly/`, and `/etc/cron.monthly/` require execute permissions.

```bash
chmod +x /etc/cron.daily/example-script
```

- Review `/var/log/messages` for cron activity:

```bash
grep cron /var/log/messages
grep CROND /var/log/messages
```

- No single command displays every scheduled cron job. Review all applicable locations:
  - User crontabs with `crontab -l`.
  - Other users' crontabs with `crontab -l -u username`.
  - `/etc/crontab`.
  - `/etc/cron.d/`.
  - `/etc/cron.hourly/`, `/etc/cron.daily/`, `/etc/cron.weekly/`, and `/etc/cron.monthly/`.

## At Service: One-Time Task Scheduling

The `at` service schedules commands to run once at a specified time.

## Managing the `atd` Service

| Command | Description |
|---|---|
| `systemctl status atd` | Check the `atd` service status. |

## `at` Commands

| Command | Description |
|---|---|
| `at 15:00` | Schedule a job for 3:00 PM. |
| `at teatime` | Schedule a job for 4:00 PM. |
| `at noon` | Schedule a job for noon. |
| `at tomorrow` | Schedule a job for tomorrow. |
| `atq` | View scheduled `at` jobs. |
| `atrm jobnumber` | Remove a scheduled job. |
| `batch` | Execute a job when system load permits; typically below `0.8`. |

## Using `at`

1. Start an `at` session with the desired execution time:

   ```bash
   at 15:00
   ```

2. Enter the command or commands to run:

   ```bash
   /path/to/script.sh
   ```

3. Press `Ctrl-D` to close the `at` prompt and save the job.

4. Verify the scheduled job:

   ```bash
   atq
   ```

Example:

```text
$ at 15:00
at> logger "One-time maintenance task started"
at> <Ctrl-D>
```

## At Service Tips

- The `atd` service must be running to schedule and execute `at` jobs.
- Ensure scripts are executable before scheduling them:

```bash
chmod +x scriptname
```