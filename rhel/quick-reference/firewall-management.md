---
layout: default
title: firewalld Management
parent: Quick Reference
grand_parent: RHEL
---

# firewalld Management

## firewalld vs. systemd Services

A **systemd service** is a running daemon or process, such as `httpd`, `sshd`, or `mariadb`.

A **firewalld service** is a firewall rule template that defines the ports, protocols, modules, and related settings needed to allow network traffic for a common application. It is separate from the systemd service that runs the actual daemon. firewalld services apply within zones and can include one or more ports or addresses. 

### Example Workflow: Apache HTTP Server

```bash
# Install the Apache HTTP Server package
dnf install httpd

# Start the daemon; the process listens on its configured port
systemctl start httpd

# Open the HTTP firewall service permanently
firewall-cmd --add-service=http --permanent

# Apply the permanent configuration to the runtime configuration
firewall-cmd --reload
```

firewalld service definitions are XML-based templates. Predefined service files commonly reside under `/usr/lib/firewalld/services/`; locally created service definitions should be managed separately from package-provided files. 

## Zones

A **zone** is a collection of firewall rules applied to incoming traffic associated with a source IP address or network interface. firewalld uses zones to assign different levels of trust to networks, interfaces, and sources. 

### Zone Assignment Priority

Traffic is evaluated according to its matching zone assignment:

1. **Source IP or network**
2. **Network interface**
3. **Default zone**

The default zone is commonly `public` unless changed.

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
| **Runtime** | Active immediately but lost after a firewalld reload or service restart. |
| **Permanent** | Saved configuration. Use `--permanent`, then reload firewalld to apply it to the active runtime configuration. |

At firewalld startup, the permanent configuration becomes the runtime configuration. A reload also replaces runtime-only settings with the permanent configuration. 

> [!IMPORTANT]
> Use `--permanent` for changes that must survive reboot or firewalld restart, then run `firewall-cmd --reload`.

## Zone Management

| Task | Command |
|---|---|
| List all zones | `firewall-cmd --get-zones` |
| Show default zone | `firewall-cmd --get-default-zone` |
| Set default zone | `firewall-cmd --set-default-zone=ZONE` |
| Show active zone configuration | `firewall-cmd --list-all` |
| Show a specific zone configuration | `firewall-cmd --list-all --zone=ZONE` |
| Show all zone configurations | `firewall-cmd --list-all-zones` |
| Show active zones with interfaces and sources | `firewall-cmd --get-active-zones` |

> [!NOTE]
> Setting the default zone with `firewall-cmd --set-default-zone` updates the default-zone setting without requiring `--permanent`. 

## Service Management

| Task | Command |
|---|---|
| List available firewalld services | `firewall-cmd --get-services` |
| List services enabled in the current zone | `firewall-cmd --list-services` |
| Show ports opened by a service | `firewall-cmd --info-service=http` |
| Add a service at runtime only | `firewall-cmd --add-service=SERVICE` |
| Add a service permanently | `firewall-cmd --add-service=SERVICE --permanent` |
| Add a service permanently to a specific zone | `firewall-cmd --add-service=SERVICE --zone=ZONE --permanent` |
| Remove a runtime service | `firewall-cmd --remove-service=SERVICE` |
| Remove a service permanently | `firewall-cmd --remove-service=SERVICE --permanent` |

## Port Management

| Task | Command |
|---|---|
| Add a port at runtime only | `firewall-cmd --add-port=8080/tcp` |
| Add a port permanently | `firewall-cmd --add-port=8080/tcp --permanent` |
| Remove a runtime port | `firewall-cmd --remove-port=8080/tcp` |
| Remove a port permanently | `firewall-cmd --remove-port=8080/tcp --permanent` |

## Interface Management

| Task | Command |
|---|---|
| Add an interface to a zone at runtime | `firewall-cmd --add-interface=eno1 --zone=dmz` |
| Add an interface to a zone permanently | `firewall-cmd --add-interface=eno1 --zone=dmz --permanent` |
| Remove an interface at runtime | `firewall-cmd --remove-interface=eno1` |
| Remove an interface permanently | `firewall-cmd --remove-interface=eno1 --permanent` |

## Source IP Management

| Task | Command |
|---|---|
| Add a source network at runtime | `firewall-cmd --add-source=192.168.1.0/24 --zone=trusted` |
| Add a source network permanently | `firewall-cmd --add-source=192.168.1.0/24 --zone=trusted --permanent` |
| Remove a source network at runtime | `firewall-cmd --remove-source=192.168.1.0/24` |
| Remove a source network permanently | `firewall-cmd --remove-source=192.168.1.0/24 --permanent` |

## Applying Changes

| Task | Command |
|---|---|
| Reload permanent configuration into the runtime configuration | `firewall-cmd --reload` |
| Restart firewalld completely | `systemctl restart firewalld` |

> [!WARNING]
> `firewall-cmd --reload` and `systemctl restart firewalld` discard runtime-only changes that were not also saved with `--permanent`. 

## Common Workflows

### Add the HTTP Service Permanently

```bash
firewall-cmd --add-service=http --permanent
firewall-cmd --reload
firewall-cmd --list-all
```

### Add a Custom TCP Port Permanently

```bash
firewall-cmd --add-port=2022/tcp --permanent
firewall-cmd --reload
firewall-cmd --list-all
```

### Allow a Specific Source Network

```bash
firewall-cmd --permanent --add-source=192.168.0.0/24 --zone=trusted
firewall-cmd --reload
```

## Best Practices

- Use `--permanent` and `firewall-cmd --reload` for persistent changes.
- Prefer predefined firewalld **services** over individual port rules when a suitable service definition exists.
- Verify firewalld is active before making changes:

  ```bash
  systemctl status firewalld
  ```

- Confirm the active zone, interfaces, sources, services, and ports after changes:

  ```bash
  firewall-cmd --get-active-zones
  firewall-cmd --list-all
  ```

- firewalld blocks traffic on ports that are not explicitly opened by a zone rule; zones such as `trusted` can allow all traffic by default. 