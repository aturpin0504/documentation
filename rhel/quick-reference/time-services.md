---
layout: default
title: Time Services
parent: Quick Reference
grand_parent: RHEL
nav_order: 12
---

# Time Synchronization with `timedatectl` and Chrony

## `timedatectl` Commands

```bash
# Display current time settings
timedatectl

# List available timezones
timedatectl list-timezones

# Set timezone
timedatectl set-timezone America/New_York

# Enable NTP synchronization
timedatectl set-ntp 1

# Convert a Unix timestamp to a human-readable date
date -d @1732032000
```

{: .note }
> The timezone is a local display setting and is normally configured once during server setup. NTP synchronizes system time; it does not change the configured timezone.

## Chrony Service Management

NTP synchronization is implemented by the `chronyd` daemon, provided by the `chrony` package.

```bash
# Check Chrony service status
systemctl status chronyd

# Enable and start Chrony
systemctl enable --now chronyd

# Restart Chrony after configuration changes
systemctl restart chronyd

# Display NTP sources and sync status
chronyc sources

# Display synchronization and clock-tracking details
chronyc tracking
```

## Configuring an NTP Client

Edit `/etc/chrony.conf` and configure your NTP server(s). STIG baselines may require `maxpoll 16` or lower on `server` or `pool` lines.

{: .note }
> Editing the existing `/etc/chrony.conf` as `root` normally preserves its ownership, permissions, and SELinux context. If the file is replaced, copied from another location, restored from backup, or created by automation, verify and restore its expected metadata before restarting `chronyd`.

```ini
# pool 2.rhel.pool.ntp.org iburst

server 0.us.pool.ntp.mil iburst maxpoll 16
```

- `iburst` accelerates initial synchronization using short polling intervals for the first several requests.
- `maxpoll 16` caps the maximum polling interval at 65,536 seconds, approximately 18.2 hours.

### Verify Configuration File Metadata

Use these commands when `/etc/chrony.conf` has been replaced or restored rather than edited in place:

```bash
# Check current ownership and numeric permissions
stat -c '%U:%G %a %n' /etc/chrony.conf

# Check the current SELinux context
ls -lZ /etc/chrony.conf

# Display the SELinux context expected for this path
matchpathcon /etc/chrony.conf

# Restore the policy-defined SELinux context if necessary
restorecon -v /etc/chrony.conf
```

The configuration file should normally remain owned by `root:root` and commonly uses mode `0644`.

```bash
# Restore typical ownership and permissions if the file was replaced
chown root:root /etc/chrony.conf
chmod 0644 /etc/chrony.conf

# Restore the SELinux context defined by local policy
restorecon -v /etc/chrony.conf
```

{: .warning }
> Prefer `restorecon` over `chcon`. `restorecon` applies the SELinux context defined by the installed policy for `/etc/chrony.conf`. A context set manually with `chcon` can be overwritten during a filesystem relabel.

Apply the configuration and verify synchronization:

```bash
systemctl restart chronyd
systemctl status chronyd --no-pager
chronyc sources
chronyc tracking
```

## Configuring an NTP Server with Upstream Synchronization

Edit `/etc/chrony.conf` to configure upstream time sources and permit NTP clients from the specified network:

```ini
# pool 2.rhel.pool.ntp.org iburst

server dc1.example.com iburst maxpoll 16
server dc2.example.com iburst maxpoll 16

allow 192.168.0.0/16
```

The `allow` directive permits NTP clients from the specified subnet to use the host as an NTP server. Without an `allow` directive, `chronyd` operates as an NTP client only.

Do not manually configure stratum for an upstream-synchronized server; Chrony calculates stratum from its selected reference source.

```bash
systemctl restart chronyd
chronyc sources
chronyc tracking
```

### Open the Firewall for NTP Clients

NTP uses UDP port `123`. Permit it only when this system serves time to other systems:

```bash
# Allow the predefined NTP service
firewall-cmd --add-service=ntp --permanent
firewall-cmd --reload

# Or explicitly allow UDP port 123
firewall-cmd --add-port=123/udp --permanent
firewall-cmd --reload
```

{: .warning }
> Do not open UDP port `123` on a system that operates only as an NTP client. Limit NTP access to approved networks and hosts.