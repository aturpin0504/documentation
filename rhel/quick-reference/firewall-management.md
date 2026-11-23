---
layout: default
title: firewalld
parent: Quick Reference
grand_parent: RHEL
nav_order: 14
---

# firewalld Management

## firewalld vs. systemd Services

A **systemd service** is a running daemon or process (such as `httpd`, `sshd`, or `mariadb`). A **firewalld service** is an XML-based firewall rule template that defines the ports, protocols, and optional modules to allow for an application within a zone.

Packaged firewalld service definitions are typically stored in `/usr/lib/firewalld/services/`. Locally created or overridden service definitions are stored in `/etc/firewalld/services/`.

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

# Set the default zone
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

## Custom Firewalld Services

Create a custom firewalld service when an application uses nonstandard ports or when multiple ports should be managed as one named rule.

{: .important }
> Creating a service definition does **not** open its ports. After creating or modifying the definition, add the service to the required zone.

### Create and Enable a Custom Service

```bash
# Create a new permanent service definition
firewall-cmd --permanent --new-service=myapp

# Add a short name and description
firewall-cmd --permanent --service=myapp --set-short="My Application"
firewall-cmd --permanent --service=myapp \
  --set-description="Allows access to My Application."

# Add one or more TCP or UDP ports to the named service
firewall-cmd --permanent --service=myapp --add-port=8443/tcp
firewall-cmd --permanent --service=myapp --add-port=8444/tcp
firewall-cmd --permanent --service=myapp --add-port=5353/udp

# Load the permanent service definition into the runtime configuration
firewall-cmd --reload

# Enable the named service in the intended zone
firewall-cmd --permanent --zone=public --add-service=myapp
firewall-cmd --reload

# Verify the service definition and zone membership
firewall-cmd --info-service=myapp
firewall-cmd --zone=public --list-services
```

### Modify a Custom Service

```bash
# Display the current custom service definition
firewall-cmd --info-service=myapp

# Add another port to the service
firewall-cmd --permanent --service=myapp --add-port=8445/tcp

# Remove a port that is no longer required
firewall-cmd --permanent --service=myapp --remove-port=8444/tcp

# Update the service description
firewall-cmd --permanent --service=myapp \
  --set-description="Allows HTTPS and application traffic."

# Apply and verify the updated definition
firewall-cmd --reload
firewall-cmd --info-service=myapp
```

### Delete a Custom Service

```bash
# Remove the service from each zone where it is enabled
firewall-cmd --permanent --zone=public --remove-service=myapp

# Delete the permanent custom service definition
firewall-cmd --permanent --delete-service=myapp

# Apply the deletion to the runtime configuration
firewall-cmd --reload

# Confirm that the service is no longer available
firewall-cmd --get-services
```

{: .warning }
> Do not modify packaged service files under `/usr/lib/firewalld/services/`. Package updates can replace those files. Create a custom service definition under `/etc/firewalld/services/` instead.

## Port Management

Use individual port rules when a predefined or custom named service is not appropriate.

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

# 3. Create a named custom service and open it in the public zone
firewall-cmd --permanent --new-service=myapp
firewall-cmd --permanent --service=myapp --add-port=8443/tcp
firewall-cmd --permanent --service=myapp --add-port=8444/tcp
firewall-cmd --reload
firewall-cmd --permanent --zone=public --add-service=myapp
firewall-cmd --reload
firewall-cmd --zone=public --list-services

# 4. Add a custom TCP port permanently and reload
firewall-cmd --add-port=2022/tcp --permanent
firewall-cmd --reload
firewall-cmd --list-all

# 5. Allow a specific source network in the trusted zone
firewall-cmd --permanent --add-source=192.168.0.0/24 --zone=trusted
firewall-cmd --reload
firewall-cmd --get-active-zones
```