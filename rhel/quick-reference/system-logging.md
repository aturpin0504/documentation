---
layout: default
title: System Logging
parent: Quick Reference
grand_parent: RHEL
nav_order: 10
---

# System Logging Services

## Logging Services Overview

| Service | Description |
|---|---|
| `rsyslogd` | Enhanced syslog daemon that writes persistent text log files under `/var/log`. |
| `systemd-journald` | `systemd` integrated logging service that stores messages in binary journal format. |
| `auditd` | Auditing service for detailed event tracing; SELinux audit events are recorded through the Linux audit subsystem. |

### When to Use Each Logging Tool

| Tool | Primary Use Case |
|---|---|
| `journalctl` | Troubleshooting services, boot issues, and real-time system events. |
| `/var/log/*` files | Persistent logs, scripts, and text processing with tools such as `grep`, `awk`, and `sed`. |
| `ausearch` / `aureport` | Security events, SELinux denials, compliance auditing, and STIG-related review. |

1. **`systemd-journald`** collects messages from `systemd` services, kernel messages, and application `stdout`/`stderr`.
2. **`rsyslog`** receives messages forwarded from `journald` and writes them to persistent text logs under `/var/log`.
3. **`auditd`** operates through the kernel audit subsystem and records audit events in `/var/log/audit/audit.log`.

## Common Log Files

| Log File | Contents |
|---|---|
| `/var/log/messages` | Generic system log; many system messages are written here. |
| `/var/log/secure` | Authentication-related messages and errors. |
| `/var/log/audit/audit.log` | Linux audit messages, including SELinux audit events. |
| `/var/log/boot.log` | System startup messages. |
| `/var/log/dmesg` | Kernel log messages. |
| `/var/log/maillog` | Mail-related messages. |
| `/var/log/cron` | Cron job messages. |
| `/var/log/httpd/` | Apache HTTP Server logs. |

## Log File Monitoring

```bash
# Monitor a log file in real time
tail -f /var/log/messages

# Show the last 20 lines of a log file
tail -20 /var/log/secure

# View a log file with a pager
less /var/log/messages

# Search a log file
grep "Failed password" /var/log/secure
```

## `logger`: Write Messages to Syslog

```bash
# Write a message to syslog
logger "Maintenance task completed"

# Log using a specific facility and priority
logger -p user.notice "Custom application status message"
```

## `journalctl`: Query the System Journal

### Common `journalctl` Options

| Option | Purpose |
|---|---|
| `-f` | Follow new messages in real time |
| `-n <num>` | Show the last `<num>` lines |
| `-p <priority>` | Filter by priority, such as `err`, `warning`, or `crit` |
| `-u <unit>` | Show messages for a specific `systemd` unit |
| `--since` / `--until` | Filter by relative or absolute time |
| `-k` | Show kernel messages |
| `-b` / `-b -1` | Show messages from the current boot or previous boot |
| `--disk-usage` | Show disk space used by persistent and runtime journals |
| `--flush` | Move volatile journal entries to persistent storage when configured |

```bash
# View the entire journal
journalctl

# Follow new messages in real time
journalctl -f

# Follow new messages for a specific service
journalctl -u sshd.service -f

# Show the last 20 lines
journalctl -n 20

# Show only error-level messages
journalctl -p err

# Show errors since the current boot
journalctl -b -p err

# Show kernel messages
journalctl -k

# Show messages from the previous boot
journalctl -b -1

# Filter by time
journalctl --since yesterday
journalctl --since today
journalctl --since "2019-06-08 09:00:00"
journalctl --since "09:00:00" --until "15:00:00"

# Display journal disk usage
journalctl --disk-usage
```

## Making `journald` Persistent

Persistent journal storage saves logs under `/var/log/journal/` so they remain available after a reboot.

{: .note }
> Verify the local configuration and organizational logging baseline before making changes. Persistent storage may be configured through the default `Storage=auto` behavior, an explicit `Storage=persistent` setting, or an approved configuration-management process.

### Verify Current Configuration

```bash
# Check for an explicit Storage setting in the main configuration file
grep -E '^[[:space:]]*Storage=' /etc/systemd/journald.conf

# Check for Storage settings in configuration drop-in files
grep -R -E '^[[:space:]]*Storage=' \
  /etc/systemd/journald.conf.d/ 2>/dev/null

# Check whether the persistent journal directory exists,
# including ownership, permissions, and SELinux context
ls -ldZ /var/log/journal

# Display current journal disk use
journalctl --disk-usage
```

### Enable Persistent Storage

Prefer a configuration drop-in rather than editing the primary `/etc/systemd/journald.conf` file directly.

```bash
# Create the journald drop-in directory if needed
mkdir -p /etc/systemd/journald.conf.d

# Configure persistent journal storage
cat > /etc/systemd/journald.conf.d/60-persistent-storage.conf <<'EOF'
[Journal]
Storage=persistent
EOF
```

Create the persistent journal directory and apply the system-defined ownership and permissions:

```bash
# Create the persistent journal directory
mkdir -p /var/log/journal

# Apply the system-defined directory configuration
systemd-tmpfiles --create --prefix /var/log/journal
```

### Verify Ownership, Permissions, and SELinux Context

```bash
# Display owner, group, numeric mode, and SELinux context
ls -ldZ /var/log/journal

# Display the SELinux context expected for this path
matchpathcon /var/log/journal

# Restore the policy-defined SELinux context recursively if needed
restorecon -RFv /var/log/journal
```

{: .note }
> Do not normally set ownership, mode, SGID, or SELinux labels manually for `/var/log/journal`. `systemd-tmpfiles` applies the system-defined directory configuration, and `restorecon` applies the SELinux context defined by the local policy.

### Apply and Verify Persistent Logging

```bash
# Restart journald to apply configuration changes
systemctl restart systemd-journald

# Flush volatile journal data to persistent storage
journalctl --flush

# Verify persistent journal files exist
find /var/log/journal -type f -name '*.journal' -ls

# Display journal disk usage
journalctl --disk-usage

# View logs from the current boot
journalctl -b
```

{: .warning }
> Do not use `chcon` as the normal method for labeling `/var/log/journal`. Use `restorecon` so the directory receives the label defined by the installed SELinux policy. A manual `chcon` label can be lost during a filesystem relabel.

## Audit Log Commands

```bash
# Search for SELinux denials
ausearch -m AVC
ausearch -m AVC -ts recent -i

# Search for login attempts
ausearch -m USER_LOGIN
ausearch -m USER_LOGIN -ts today -i
ausearch -ts recent -m USER_LOGIN -i

# Search access to a specific file
ausearch -f /etc/passwd -i

# Search activity by UID
ausearch -ui 1000

# Show today's or recent audit events
ausearch -ts today
ausearch -ts recent -i

# Generate summary, authentication, failed, or executable reports
aureport
aureport --auth
aureport --auth --failed
aureport --failed
aureport -x --summary
```

## `rsyslogd` Configuration

| File or Directory | Purpose |
|---|---|
| `/etc/rsyslog.conf` | Main `rsyslogd` configuration file. |
| `/etc/rsyslog.d/` | Directory for custom `rsyslogd` configuration files. |

### `rsyslogd` Facilities

| Facility | Common Use |
|---|---|
| `auth` / `authpriv` | Authentication-related messages. |
| `cron` | Cron service messages. |
| `daemon` | Generic daemon messages. |
| `kern` | Kernel messages. |
| `mail` | Email-related messages. |
| `user` | User-space application messages. |
| `local0` through `local7` | Custom application or organizational facility assignments. |

### `rsyslogd` Priorities

| Priority | Description |
|---|---|
| `debug` | Debug messages; maximum detail. |
| `info` | Informational messages for normal operation. |
| `notice` | Normal but potentially noteworthy messages. |
| `warning` | Suboptimal condition without a confirmed error. |
| `err` | Non-critical error. |
| `crit` | Critical error. |
| `alert` | Service availability may be ending. |
| `emerg` | Service availability has ended or the system is unusable. |

### `rsyslog.conf` Rule Examples

```rsyslog
# Write all authentication and authorization messages to /var/log/secure
authpriv.* /var/log/secure

# Write all mail messages to /var/log/maillog
mail.* /var/log/maillog

# Write all cron messages to /var/log/cron
cron.* /var/log/cron

# Write local1 facility errors to a custom log
local1.error /var/log/custom.log
```

Restart `rsyslog` after modifying its configuration:

```bash
systemctl restart rsyslog
```

## Log Rotation with `logrotate`

| Configuration | Purpose |
|---|---|
| `/etc/logrotate.conf` | Main `logrotate` configuration file. |
| `/etc/logrotate.d/` | Service-specific log rotation settings. |
