---
layout: default
title: Time Services
parent: Quick Reference
grand_parent: RHEL
nav_order: 11
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
```

## Configuring an NTP Client

Edit `/etc/chrony.conf` and configure your NTP server(s) (STIG baselines typically require `maxpoll 16` or lower on `server`/`pool` lines):

```ini
# pool 2.rhel.pool.ntp.org iburst

server 0.us.pool.ntp.mil iburst maxpoll 16
```

- `iburst` accelerates initial synchronization using short polling intervals for the first several requests.
- `maxpoll 16` caps the maximum polling interval at 65,536 seconds (~18.2 hours) to meet STIG requirements.

Apply the configuration and verify synchronization:

```bash
systemctl restart chronyd
chronyc sources
```

## Configuring an NTP Server with Upstream Synchronization

Edit `/etc/chrony.conf` to configure upstream time sources and permit NTP clients from the specified network:

```ini
# pool 2.rhel.pool.ntp.org iburst

server dc1.example.com iburst maxpoll 16
server dc2.example.com iburst maxpoll 16

allow 192.168.0.0/16
```

The `allow` directive permits NTP clients from the specified subnet to use the host as an NTP server. Without an `allow` directive, `chronyd` operates as an NTP client only. Do not manually configure stratum for an upstream-synchronized server; Chrony calculates it from its selected reference source.

```bash
systemctl restart chronyd
chronyc sources
```

### Open the Firewall for NTP Clients

NTP uses UDP port `123`; permit it only when this system serves time to other systems:

```bash
firewall-cmd --add-service=ntp --permanent
firewall-cmd --reload

# Or explicitly open UDP port 123
firewall-cmd --add-port=123/udp --permanent
firewall-cmd --reload
```