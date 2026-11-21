---
layout: default
title: firewalld
parent: Quick Reference
grand_parent: RHEL
nav_order: 14
---

# firewalld Management

## firewalld vs. systemd Services

A **systemd service** is a running daemon or process (such as `httpd`, `sshd`, or `mariadb`). A **firewalld service** is an XML-based firewall rule template (predefined under `/usr/lib/firewalld/services/`) that defines the ports, protocols, and modules to allow for an application within a zone.

### Example Workflow: Apache HTTP Server

```bash
# Install and start the Apache HTTP daemon
dnf install httpd
systemctl start httpd

# Open the HTTP firewalld service permanently and apply to runtime
firewall-cmd --add-service=http --permanent
firewall-cmd --reload
```

## Zones

A **zone** is a collection of firewall rules applied to incoming traffic based on:

1. **Source IP or network** (highest priority)
2. **Network interface**
3. **Default zone** (commonly `public`)

### Common Zones

| Zone | Typical Use Case |
|---|---|
| `drop` | Drops packets without a reply. |
| `block` | Rejects connections with an ICMP host-prohibited response. |
| `public` | Default zone for untrusted networks with limited access. |
| `dmz` | DMZ servers with selected incoming services allowed. |
| `work` | Work networks where most systems are trusted. |
| `home` | Home networks where most systems are trusted. |
| `internal` | Internal networks with a higher level of trust. |
| `trusted` | Accepts all connections. |
| `external` | Supports NAT masquerading for routers or external-facing interfaces. |

## Runtime vs. Persistent Configuration

| Configuration Type | Behavior |
|---|---|
| **Runtime** | Active immediately, but lost after `firewall-cmd --reload` or `systemctl restart firewalld`. |
| **Permanent** | Saved configuration (`--permanent`). Run `firewall-cmd --reload` to apply it to the active runtime configuration. |

{: .important }
> Use `--permanent` for changes that must survive reboot or firewalld restart, then run `firewall-cmd --reload`.

## Zone Management

```bash
# List all zones and check the default zone
firewall-cmd --get-zones
firewall-cmd --get-default-zone

# Set the default zone (persists without --permanent)
firewall-cmd --set-default-zone=ZONE

# Inspect active, specific, or all zone configurations
firewall-cmd --list-all
firewall-cmd --list-all --zone=ZONE
firewall-cmd --list-all-zones

# Show active zones with their interfaces and sources
firewall-cmd --get-active-zones
```

## Service Management

Prefer predefined firewalld **services** over individual port rules when a suitable service definition exists.

```bash
# List available services, enabled services, or ports opened by a service
firewall-cmd --get-services
firewall-cmd --list-services
firewall-cmd --info-service=http

# Add a service (runtime only vs. permanent)
firewall-cmd --add-service=SERVICE
firewall-cmd --add-service=SERVICE --permanent
firewall-cmd --add-service=SERVICE --zone=ZONE --permanent

# Remove a service (runtime only vs. permanent)
firewall-cmd --remove-service=SERVICE
firewall-cmd --remove-service=SERVICE --permanent
```

## Port Management

```bash
# Add a port (runtime only vs. permanent)
firewall-cmd --add-port=8080/tcp
firewall-cmd --add-port=8080/tcp --permanent

# Remove a port (runtime only vs. permanent)
firewall-cmd --remove-port=8080/tcp
firewall-cmd --remove-port=8080/tcp --permanent
```

## Interface and Source IP Management

```bash
# Assign an interface to a zone (runtime vs. permanent)
firewall-cmd --add-interface=eno1 --zone=dmz
firewall-cmd --add-interface=eno1 --zone=dmz --permanent

# Remove an interface from its zone
firewall-cmd --remove-interface=eno1
firewall-cmd --remove-interface=eno1 --permanent

# Assign a source network to a zone (runtime vs. permanent)
firewall-cmd --add-source=192.168.1.0/24 --zone=trusted
firewall-cmd --add-source=192.168.1.0/24 --zone=trusted --permanent

# Remove a source network
firewall-cmd --remove-source=192.168.1.0/24
firewall-cmd --remove-source=192.168.1.0/24 --permanent
```

## Applying Changes

```bash
# Reload permanent configuration into runtime
firewall-cmd --reload

# Restart firewalld completely
systemctl restart firewalld
```

{: .warning }
> `firewall-cmd --reload` and `systemctl restart firewalld` discard runtime-only changes that were not saved with `--permanent`.

## Common Workflows

```bash
# 1. Verify firewalld is active
systemctl status firewalld

# 2. Add HTTP permanently, reload, and verify
firewall-cmd --add-service=http --permanent
firewall-cmd --reload
firewall-cmd --list-all

# 3. Add a custom TCP port permanently and reload
firewall-cmd --add-port=2022/tcp --permanent
firewall-cmd --reload
firewall-cmd --list-all

# 4. Allow a specific source network in the trusted zone
firewall-cmd --permanent --add-source=192.168.0.0/24 --zone=trusted
firewall-cmd --reload
firewall-cmd --get-active-zones
```