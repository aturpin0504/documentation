---
layout: default
title: SSH Basic Hardening
parent: Quick Reference
grand_parent: RHEL
---

# SSH Basic Hardening

## SSH Server Configuration

The main SSH server configuration file is:

```text
/etc/ssh/sshd_config
```

On RHEL 9 and later, configuration drop-in files under `/etc/ssh/sshd_config.d/` are also supported and can be preferable for locally managed settings. 

Edit the SSH server configuration:

```bash
vim /etc/ssh/sshd_config
```

### Common Hardening Settings

| Objective | `sshd_config` Setting |
|---|---|
| Disable direct root login | `PermitRootLogin no` |
| Restrict SSH access to named users | `AllowUsers user1 user2 user3` |
| Disable password authentication and require keys | `PasswordAuthentication no` |

Example:

```ini
PermitRootLogin no
AllowUsers user1 user2 user3
PasswordAuthentication no
```

`PermitRootLogin`, `PasswordAuthentication`, and `AllowUsers` can be used to restrict root access, disable password-based login, and limit which users may connect. 

After modifying SSH configuration:

```bash
# Validate configuration syntax
sshd -t

# Restart SSH service
systemctl restart sshd
```

> [!WARNING]
> Keep an existing SSH session open while changing SSH configuration. Test a new connection before closing the original session.

## Changing the SSH Port

### Configure a New Port

Add a `Port` directive to SSH server configuration:

```ini
Port 2022
```

Then label the port for SELinux:

```bash
semanage port -a -t ssh_port_t -p tcp 2022
```

Open the port in firewalld:

```bash
firewall-cmd --add-port=2022/tcp --permanent
firewall-cmd --reload
```

Restart SSH:

```bash
systemctl restart sshd
```

RHEL documentation describes adding the `ssh_port_t` SELinux label for a non-standard SSH port, opening the corresponding firewalld port, and applying the firewall configuration. 

### Port Change Tips

- Keep at least two SSH sessions open while changing ports.
- Test connectivity to the new port before closing existing sessions.
- Use `semanage port -m` to modify an existing SELinux port label rather than adding a duplicate label.

Example:

```bash
semanage port -m -t ssh_port_t -p tcp 443
```

## Manual SSH Key Installation

### Create the `.ssh` Directory

```bash
mkdir -p ~/.ssh
chmod 700 ~/.ssh
```

### Add the Public Key

Edit the authorized key file:

```bash
vim ~/.ssh/authorized_keys
```

Paste the complete public key on a single line, save the file, and set restrictive permissions:

```bash
chmod 600 ~/.ssh/authorized_keys
```

### Windows Client Workflow

For SecureCRT, PuTTY, MobaXterm, or similar clients:

1. Generate an RSA key pair in the Windows SSH client.
2. Export or copy the public key.
3. Connect to the RHEL server using authorized password authentication.
4. Add the public key to `~/.ssh/authorized_keys`.
5. Set permissions:

   ```bash
   chmod 700 ~/.ssh
   chmod 600 ~/.ssh/authorized_keys
   ```

6. Test key-based authentication in a separate SSH session.
7. Disable password authentication only after key-based login has been verified.

> [!NOTE]
> On FIPS-enabled systems, do not use Ed25519 keys. Use RSA or another organization-approved FIPS-compatible key type. Red Hat documentation specifically identifies RSA and ECDSA as alternatives when Ed25519 is not appropriate for FIPS-based deployments. 

### Common SSH Key Permission Issues

| Item | Required Permissions |
|---|---|
| `~/.ssh/` | `700` (`drwx------`) |
| `~/.ssh/authorized_keys` | `600` (`-rw-------`) |

Additional requirements:

- Both the `.ssh` directory and `authorized_keys` file must be owned by the target user.
- They must not be owned by `root`.
- Incorrect ownership or permissions can cause SSH to reject key-based authentication or fall back to password authentication.

## SSH Troubleshooting

| Task | Command |
|---|---|
| Enable verbose client-side debugging | `ssh -vvv user@server2` |
| View SSH service logs | `journalctl -u sshd` |
| Follow SSH logs in real time | `journalctl -u sshd -f` |
| Show SSH logs since last boot | `journalctl -u sshd -b` |
| Show SSH logs from the past hour | `journalctl -u sshd --since "1 hour ago"` |
| Monitor authentication log on applicable systems | `tail -f /var/log/secure` |
| Check SELinux SSH port labels | `semanage port -l \| grep ssh` |
| Verify SSH is listening on port 22 | `ss -tlnp \| grep :22` |
| Verify SSH is listening on port 2022 | `ss -tlnp \| grep :2022` |

For STIG-hardened systems, use `journalctl -u sshd` and the audit logs as the primary sources for SSH troubleshooting when authentication events are not written to `/var/log/secure`.

## Useful `sshd_config` Options

| Option | Purpose |
|---|---|
| `Port` | Changes the SSH listening port from the default port 22. |
| `PermitRootLogin` | Controls direct root login. Set to `no` to require administrators to authenticate as another user and elevate with `sudo`. |
| `AllowUsers` | Whitelist of users permitted to connect through SSH. Users are space-separated. |
| `PasswordAuthentication` | Controls password-based SSH authentication. Set to `no` to require key-based authentication. |
| `ClientAliveInterval` | Number of seconds between server-to-client keepalive packets. |
| `ClientAliveCountMax` | Number of unanswered keepalive packets allowed before disconnecting the client. |
| `UseDNS` | Controls reverse DNS lookup of connecting client IP addresses. |
| `GSSAPIAuthentication` | Controls Kerberos/GSSAPI authentication. |

### Keepalive Example

```ini
ClientAliveInterval 60
ClientAliveCountMax 3
```

This configuration disconnects an unresponsive client after approximately 180 seconds, or three minutes.

## Quick Tips

- Main configuration file: `/etc/ssh/sshd_config`.
- RHEL 9+ supports configuration drop-ins under `/etc/ssh/sshd_config.d/`. 
- Validate SSH configuration with `sshd -t` before restarting the service.
- Restart or reload `sshd` after configuration changes.
- Set SELinux port labels before restarting SSH when using non-standard ports.
- Keep a backup SSH session open before applying SSH changes.
- Test key-based authentication before disabling `PasswordAuthentication`.
