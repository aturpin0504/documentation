---
layout: default
title: SSH & SFTP
parent: Quick Reference
grand_parent: RHEL
nav_order: 13
---

# SSH Management and Hardening

## SSH Service and Key Files

```bash
# Manage the SSH daemon
systemctl status sshd
systemctl start sshd
systemctl enable sshd
systemctl reload sshd
systemctl restart sshd
```

| File | Purpose |
|---|---|
| `/etc/ssh/sshd_config` | Main SSH server configuration file (`sshd_config.d/` for drop-ins where supported). |
| `/etc/ssh/ssh_config` | System-wide SSH client configuration. |
| `~/.ssh/config` | Per-user SSH client configuration. |
| `~/.ssh/known_hosts` | Server fingerprints for previously connected hosts. |
| `~/.ssh/authorized_keys` | Public keys authorized to authenticate to the account. |

## Login Messages

| File | Display Time | Description |
|---|---|---|
| `/etc/motd` | After login | Message of the Day; displayed for shell logins. |
| `/etc/issue` | Before login | Console login message. |
| `/etc/issue.net` | Before login | SSH or network login message. |

## SSH Server Configuration

The main SSH server configuration file is `/etc/ssh/sshd_config`. Where supported, drop-in files under `/etc/ssh/sshd_config.d/` are preferable for locally managed settings.

### Useful `sshd_config` Options

| Option | Purpose |
|---|---|
| `Port` | Changes the SSH listening port from the default port 22. |
| `PermitRootLogin` | Controls direct root login. Set to `no` to require authenticating as another user and elevating with `sudo`. |
| `AllowUsers` | Space-separated allowlist of users permitted to connect through SSH. |
| `PasswordAuthentication` | Controls password-based SSH authentication. Set to `no` to require key-based authentication. |
| `ClientAliveInterval` | Number of seconds between server-to-client keepalive packets. |
| `ClientAliveCountMax` | Number of unanswered keepalive packets allowed before disconnecting the client. |
| `UseDNS` | Controls reverse DNS lookup of connecting client IP addresses. |
| `GSSAPIAuthentication` | Controls Kerberos/GSSAPI authentication. |

Example hardening settings in `/etc/ssh/sshd_config`:

```ini
PermitRootLogin no
AllowUsers user1 user2 user3
PasswordAuthentication no

# Disconnect unresponsive clients after ~180 seconds (60s * 3)
ClientAliveInterval 60
ClientAliveCountMax 3
```

After modifying SSH configuration, validate syntax and restart:

```bash
sshd -t
systemctl restart sshd
```

{: .warning }
> Keep an existing SSH session open while changing SSH configuration. Test a new connection before closing the original session.

## Changing the SSH Port

1. Add a `Port` directive to `/etc/ssh/sshd_config` (for example, `Port 2022`).
2. Label the port for SELinux, open it in `firewalld`, and restart `sshd`:

```bash
# Add SELinux port label (use -m instead of -a to modify an existing label, e.g., port 443)
semanage port -a -t ssh_port_t -p tcp 2022

# Open the port in firewalld
firewall-cmd --add-port=2022/tcp --permanent
firewall-cmd --reload

# Validate and restart sshd
sshd -t
systemctl restart sshd
```

Keep at least two SSH sessions open and test connectivity to the new port before closing existing sessions.

## Manual SSH Key Installation

### Configure `.ssh` and `authorized_keys`

```bash
mkdir -p ~/.ssh
chmod 700 ~/.ssh

# Paste the complete public key on a single line, then set restrictive permissions
vim ~/.ssh/authorized_keys
chmod 600 ~/.ssh/authorized_keys
```

### Windows Client Workflow

For SecureCRT, PuTTY, MobaXterm, or similar clients:

1. Generate an RSA key pair in the Windows SSH client and export/copy the public key.
2. Connect to the RHEL server using authorized password authentication.
3. Add the public key to `~/.ssh/authorized_keys` and set permissions (`chmod 700 ~/.ssh` and `chmod 600 ~/.ssh/authorized_keys`).
4. Test key-based authentication in a separate SSH session before disabling `PasswordAuthentication`.

{: .note }
> On FIPS-enabled systems, do not use Ed25519 keys. Use RSA or ECDSA (or another organization-approved FIPS-compatible key type).

### Required SSH Key Permissions

| Item | Required Permissions |
|---|---|
| `~/.ssh/` | `700` (`drwx------`) |
| `~/.ssh/authorized_keys` | `600` (`-rw-------`) |

Both `~/.ssh/` and `authorized_keys` must be owned by the target user (not `root`), or SSH will reject key-based authentication.

## SSH Troubleshooting

```bash
# Enable verbose client-side debugging
ssh -vvv user@server2

# View and follow SSH service logs
journalctl -u sshd
journalctl -u sshd -f
journalctl -u sshd -b
journalctl -u sshd --since "1 hour ago"

# Monitor authentication log on applicable systems
tail -f /var/log/secure

# Check SELinux SSH port labels
semanage port -l | grep ssh

# Verify SSH is listening on port 22 or 2022
ss -tlnp | grep :22
ss -tlnp | grep :2022

# Check firewall rules and IPv4 addresses
firewall-cmd --list-all
ip a | grep 'inet '
```

For STIG-hardened systems, use `journalctl -u sshd` and audit logs when authentication events are not written to `/var/log/secure`.

## Blocking SSH Key Access

```bash
# Remove a specific public key entry
sed -i '/user@host/d' ~/.ssh/authorized_keys

# Fully lock an account (usermod -L alone does not block SSH keys)
usermod -L -e 1 -s /sbin/nologin username

# Remove or disable the authorized_keys file
rm ~/.ssh/authorized_keys
mv ~/.ssh/authorized_keys ~/.ssh/authorized_keys.disabled
```

You can also restrict SSH access in `/etc/ssh/sshd_config` using `DenyUsers username` or `AllowUsers user1 user2`, then validate and reload (`sshd -t && systemctl reload sshd`).

## SFTP File Transfer

Inside an `sftp` session, commands beginning with `l` operate on the local workstation rather than the remote server.

| Task | SFTP Command |
|---|---|
| Upload a file to the remote system | `put file` |
| Download a file from the remote system | `get file` |
| Download a directory recursively | `get -r directory` |
| List the remote or local directory | `ls` / `lls` |
| Change the remote or local directory | `cd dir` / `lcd dir` |
| Show the remote or local working directory | `pwd` / `lpwd` |
| Exit the SFTP session | `quit` |