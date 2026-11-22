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

# Show listening TCP/UDP sockets with numeric ports and owning processes
ss -tulnp

# Show all sockets (listening + active/established connections)
ss -tulna
```

A **socket** is a network communication endpoint—the combination of protocol (`TCP`/`UDP`), IP address, and port number (plus state such as `LISTEN` or `ESTAB`). Most daily troubleshooting only needs **`-l`** (listening ports) to verify a service is bound and listening; use **`-a`** only when you also need to inspect active client connections.

### `ss` Options

| Option | Description |
|---|---|
| `-t` | Show TCP sockets |
| `-u` | Show UDP sockets |
| `-l` | Show listening sockets only |
| `-n` | Do not resolve service or host names |
| `-p` | Show the process using the socket |
| `-a` | Show all sockets (listening and non-listening/established) |

{: .warning }
> Do not use `ifconfig`. Use the `ip` command instead.

## NetworkManager (`nmcli`)

```bash
# Check NetworkManager status
systemctl status NetworkManager

# List connection profiles and device status
nmcli con show
nmcli dev status

# Inspect all settings on a connection profile
nmcli con show ens192

# Apply connection changes without restarting the interface
nmcli device reapply ens192
```

### Changing IP, Subnet Mask, Gateway, and DNS

Use CIDR notation for the subnet mask (for example, `/24` for `255.255.255.0`, `/16` for `255.255.0.0`). Stage all changes with `nmcli con mod` first (your current SSH session stays active), then activate in one step with `nmcli con up` (do **not** run `nmcli con down` first):

```bash
# 1. Stage IP, subnet mask (/24), gateway, and DNS servers (replace ens192 with your connection name)
nmcli con mod ens192 \
  ipv4.addresses 10.0.0.50/24 \
  ipv4.gateway 10.0.0.1 \
  ipv4.dns "10.0.0.10 10.0.0.11" \
  ipv4.method manual

# Optional: add, remove, or set DNS search domains without overwriting other settings
nmcli con mod ens192 +ipv4.dns 10.0.0.12
nmcli con mod ens192 -ipv4.dns 10.0.0.12
nmcli con mod ens192 ipv4.dns-search "example.com"

# 2. Activate the new configuration (current SSH session will drop if the IP changed; reconnect on the new IP)
nmcli con up ens192
```

{: .note }
> Use the VMware console if you are also changing the VM's VLAN/port group in vSphere, or if a typo in the IP/gateway leaves the VM unreachable.

{: .important }
> RHEL's resolver (`glibc`) supports a **maximum of 3 `nameserver` entries** in `/etc/resolv.conf`. Any DNS servers configured beyond the first 3 are silently ignored.

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
> Do not edit `/etc/resolv.conf` directly. NetworkManager can overwrite the file. Use `nmcli` to manage DNS settings.