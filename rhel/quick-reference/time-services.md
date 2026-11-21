---
layout: default
title: Time Synchronization with timedatectl and Chrony
parent: Quick Reference
grand_parent: RHEL
nav_order: 11
---

# Time Synchronization with `timedatectl` and Chrony

## `timedatectl` Commands

| Task | Command |
|---|---|
| Display current time settings | `timedatectl` |
| Set timezone | `timedatectl set-timezone America/New_York` |
| List available timezones | `timedatectl list-timezones` |
| Enable NTP synchronization | `timedatectl set-ntp 1` |
| Convert a Unix timestamp to a date | `date -d @<unix_timestamp>` |

Example:

```bash
timedatectl set-timezone America/New_York
timedatectl set-ntp 1
date -d @1732032000
```

{: .note }
> The timezone is a local display setting and is normally configured once during server setup. NTP synchronizes system time; it does not change the configured timezone.

## Chrony Service Management

Starting with RHEL 8, NTP synchronization is implemented by the `chronyd` daemon, provided by the `chrony` package. 

| Task | Command |
|---|---|
| Check Chrony service status | `systemctl status chronyd` |
| Enable and start Chrony | `systemctl enable --now chronyd` |
| Restart Chrony | `systemctl restart chronyd` |
| Display NTP sources | `chronyc sources` |

## Configuring an NTP Client

Edit the Chrony configuration file:

```bash
vim /etc/chrony.conf
```

Add a custom NTP server:

```ini
# pool 2.rhel.pool.ntp.org iburst

server server1.example.com iburst
```

Apply the configuration and verify synchronization:

```bash
systemctl restart chronyd
chronyc sources
```

The `iburst` option accelerates the initial synchronization process by using short polling intervals for the first several requests. 

## Configuring an NTP Server with Upstream Synchronization

Edit the Chrony configuration file:

```bash
vim /etc/chrony.conf
```

Configure upstream time sources and permit NTP clients from the specified network:

```ini
# pool 2.rhel.pool.ntp.org iburst

server dc1.example.com iburst
server dc2.example.com iburst

allow 192.168.0.0/16
```

Apply the configuration:

```bash
systemctl restart chronyd
```

Verify upstream synchronization:

```bash
chronyc sources
```

The `allow` directive permits NTP clients from the specified subnet to use the host as an NTP server. Without an `allow` directive, `chronyd` normally operates as an NTP client only. 

### Open the Firewall for NTP Clients

```bash
firewall-cmd --add-service=ntp --permanent
firewall-cmd --reload
```

Alternatively, explicitly open the standard NTP port:

```bash
firewall-cmd --add-port=123/udp --permanent
firewall-cmd --reload
```

NTP uses UDP port `123`; permit that port when this system serves time to other systems. 

## Key Points

- Use `timedatectl` as the primary tool for viewing time settings, managing timezone, and enabling time synchronization.
- Configure the timezone once during initial server setup.
- NTP synchronizes system time and does not alter the configured timezone.
- Use `iburst` with `server` or `pool` directives for faster initial synchronization.
- Do not manually configure stratum for a normal upstream-synchronized NTP server; Chrony calculates it from its selected reference source.
- Open the firewall for NTP only when the system must serve time to other systems.
