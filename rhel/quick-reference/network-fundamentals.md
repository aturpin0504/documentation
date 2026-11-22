---
layout: default
title: Network Fundamentals
parent: Quick Reference
grand_parent: RHEL
nav_order: 12
---

# Network Fundamentals

## IP Address Types

- **IPv4:** 32-bit address format, such as `192.168.10.100`.
- **IPv6:** 128-bit address format, such as `fe80::...`.
- **Private IPv4 ranges:** `10.0.0.0/8`, `172.16.0.0/12`, `192.168.0.0/16`.

## Network Validation

```bash
# Show all IP addresses or a specific interface
ip addr show
ip addr show dev ens33

# Show link state and statistics
ip link show
ip -s link show

# Show routing table
ip route show

# Test connectivity
ping -c 4 8.8.8.8

# Show listening and established sockets
ss -tul
```

### `ss` Options

| Option | Description |
|---|---|
| `-t` | Show TCP sockets |
| `-u` | Show UDP sockets |
| `-l` | Show listening sockets |
| `-n` | Do not resolve service or host names |
| `-a` | Show all sockets |

{: .warning }
> Do not use `ifconfig`. Use the `ip` command instead.

## NetworkManager

```bash
# Check NetworkManager status
systemctl status NetworkManager

# Show current NetworkManager permissions
nmcli gen permissions

# Open the NetworkManager text user interface
nmtui

# Apply connection changes without restarting the interface (replace ens192 as needed)
nmcli device reapply ens192
```

In `nmtui`, you can **Edit a Connection**, **Activate a Connection**, or **Set System Hostname**. After modifying a connection in `nmtui`, deactivate and reactivate it to apply changes.

## Network Configuration Files

| File | Purpose |
|---|---|
| `/etc/NetworkManager/system-connections/` | NetworkManager connection keyfiles (legacy `ifcfg-*` files in `/etc/sysconfig/network-scripts/` on RHEL 8). |
| `/etc/hostname` | System hostname. |
| `/etc/hosts` | Static hostname-to-IP mappings. |
| `/etc/resolv.conf` | DNS configuration; typically auto-generated. |
| `/etc/nsswitch.conf` | Name-resolution lookup order. |

## Hostname and DNS Configuration

```bash
# Set or inspect the system hostname
hostnamectl set-hostname server1.example.com
hostnamectl status

# View DNS configuration or edit static host mappings
cat /etc/resolv.conf
vi /etc/hosts

# Test name resolution through /etc/hosts and DNS
getent hosts server1.example.com
host server1.example.com
```

### `/etc/hosts` Format

```text
IP_ADDRESS FQDN ALIAS
10.0.0.10 server1.example.com server1
```

{: .warning }
> Do not edit `/etc/resolv.conf` directly. NetworkManager can overwrite the file. Use `nmtui` or `nmcli` to manage DNS settings.