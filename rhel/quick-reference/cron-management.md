---
layout: default
title: Cron
parent: Quick Reference
grand_parent: RHEL
nav_order: 11
---

# Cron Service Management

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

## Adding Scripts to Cron Directories

Scripts placed in `/etc/cron.hourly/`, `/etc/cron.daily/`, `/etc/cron.weekly/`, or `/etc/cron.monthly/` are executed by `run-parts`.

Use a simple filename without periods or extensions. For example, use:

```text
/etc/cron.daily/example-maintenance
```

Do not use:

```text
/etc/cron.daily/example-maintenance.sh
/etc/cron.daily/example.maintenance
```

### Create a Daily Cron Script

Create the script with an interpreter line and use absolute paths for commands and files.

```bash
#!/bin/bash
#
# /etc/cron.daily/example-maintenance
#

/usr/bin/logger -t example-maintenance "Daily maintenance script started"

# Add maintenance commands here

/usr/bin/logger -t example-maintenance "Daily maintenance script completed"
```

Install the script with root ownership and executable permissions:

```bash
# Install a local script into cron.daily
install -o root -g root -m 0755 \
  ./example-maintenance \
  /etc/cron.daily/example-maintenance
```

Or, if the file already exists in `/etc/cron.daily/`:

```bash
# Set required owner and permissions
chown root:root /etc/cron.daily/example-maintenance
chmod 0755 /etc/cron.daily/example-maintenance
```

### Required Ownership and Permissions

| Setting | Recommended Value | Purpose |
|---|---|---|
| Owner | `root` | Prevents unprivileged users from modifying a root-executed script. |
| Group | `root` | Prevents group members from modifying the script. |
| Mode | `0755` | Allows `root` to modify the script and permits execution by the cron framework. |
| Filename | No periods (`.`) | Ensures compatibility with the `run-parts` filename filter. |
| Interpreter | Shebang, such as `#!/bin/bash` | Identifies the interpreter used to run the script. |

{: .warning }
> Scripts in the `/etc/cron.*` directories normally run with root privileges. Do not make these scripts writable by non-root users, groups, or service accounts. Use `root:root` ownership and avoid group-write or world-write permissions.

### SELinux Context

Verify the expected SELinux context before and after installing the script:

```bash
# Display the policy-defined context expected for this path
matchpathcon /etc/cron.daily/example-maintenance

# Display the file's current context
ls -lZ /etc/cron.daily/example-maintenance

# Restore the policy-defined context
restorecon -v /etc/cron.daily/example-maintenance
```

On a standard targeted-policy RHEL system, files under `/etc/cron.daily/` commonly use the `bin_t` file type. Do not hard-code the type with `chcon`; use `restorecon` so the system applies the context defined by the installed SELinux policy.

```bash
# Verify the context after restoration
ls -lZ /etc/cron.daily/example-maintenance
```

{: .note }
> Use `restorecon`, not `chcon`, for normal context correction. `restorecon` applies the policy-defined label and remains correct after a full filesystem relabel. `chcon` changes a label only temporarily and can be overwritten by relabeling.

### Validate Before Waiting for the Scheduled Run

```bash
# Confirm run-parts accepts the script name and would run it
run-parts --test /etc/cron.daily

# Run the script manually for functional testing
/etc/cron.daily/example-maintenance

# Review the script's owner, permissions, and SELinux context
ls -lZ /etc/cron.daily/example-maintenance

# Review cron and script log messages
grep -E 'CROND|example-maintenance' /var/log/messages
```

> `run-parts --test` validates whether `run-parts` recognizes the script name. It does not execute the script.

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
> Cron access can be controlled with `/etc/cron.allow` and `/etc/cron.deny`.
>
> - If `/etc/cron.allow` exists, only users listed in that file can use `crontab`; `/etc/cron.deny` is ignored.
> - If `/etc/cron.allow` does not exist but `/etc/cron.deny` exists, all users except those listed in `/etc/cron.deny` can use `crontab`.
> - If neither file exists, only `root` can use `crontab`.

## Cron Operational Checks

- Allow a three-minute margin between saving a cron configuration and the intended execution time.
- Scripts in `/etc/cron.hourly/`, `/etc/cron.daily/`, `/etc/cron.weekly/`, and `/etc/cron.monthly/` must be executable and should use simple filenames without periods (`.`) or extensions such as `.sh`. Use `run-parts --test <directory>` to confirm that the local `run-parts` implementation accepts the script.
- Use `root:root` ownership and mode `0755` for scripts that run with root privileges.
- Use `matchpathcon` and `restorecon` to verify and apply the SELinux context expected for a cron-directory script.
- No single command displays every scheduled cron job; check user crontabs (`crontab -l`), `/etc/crontab`, `/etc/cron.d/`, and `/etc/cron.{hourly,daily,weekly,monthly}/`.

```bash
# Ensure drop-in cron scripts are executable (no .sh extension)
chmod +x /etc/cron.daily/example-script

# Preview which scripts run-parts will actually execute
run-parts --test /etc/cron.daily

# Review /var/log/messages for cron activity
grep cron /var/log/messages
grep CROND /var/log/messages
```