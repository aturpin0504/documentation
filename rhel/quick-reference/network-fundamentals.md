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
> Do not use `ifconfig` on RHEL 8+. Use the `ip` command instead.

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
| `/etc/sysconfig/network-scripts/ifcfg-*` | Network connection configuration files (RHEL 8; RHEL 9 uses `/etc/NetworkManager/system-connections/`). |
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

## Login Messages

| File | Display Time | Description |
|---|---|---|
| `/etc/motd` | After login | Message of the Day; displayed for shell logins. |
| `/etc/issue` | Before login | Console login message. |
| `/etc/issue.net` | Before login | SSH or network login message. |

## SSH Service and Key Access

```bash
# Manage the SSH daemon
systemctl status sshd
systemctl start sshd
systemctl enable sshd
systemctl restart sshd
```

| File | Purpose |
|---|---|
| `/etc/ssh/ssh_config` | System-wide SSH client configuration. |
| `~/.ssh/config` | Per-user SSH client configuration. |
| `~/.ssh/known_hosts` | Server fingerprints for previously connected hosts. |
| `~/.ssh/authorized_keys` | Public keys authorized to authenticate to the account. |

### Blocking SSH Key Access

```bash
# Remove a specific public key entry
sed -i '/user@host/d' ~/.ssh/authorized_keys

# Lock an account to prevent login
usermod -L username

# Remove or disable the authorized_keys file
rm ~/.ssh/authorized_keys
mv ~/.ssh/authorized_keys ~/.ssh/authorized_keys.disabled
```

You can also restrict SSH access in `/etc/ssh/sshd_config` using `DenyUsers username` or `AllowUsers user1 user2`, then validate and reload:

```bash
sshd -t
systemctl reload sshd
```

### SSH Troubleshooting

```bash
# Verify that sshd is running and port 22 is listening
systemctl status sshd
ss -tlnp | grep :22

# Check firewall rules
firewall-cmd --list-all

# Monitor authentication attempts and sshd logs
tail -f /var/log/secure
journalctl -u sshd -f

# Show IPv4 addresses
ip a | grep 'inet '
```

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

## `tmux`: Terminal Multiplexer

Use `tmux` to maintain persistent terminal sessions across SSH connections. Most `tmux` keyboard commands begin with the `Ctrl-b` prefix (for example, `Ctrl-b d` detaches).

```bash
# Install tmux
yum install -y tmux

# Start a new unnamed or named session
tmux
tmux new -s mysession

# List all sessions
tmux ls

# Attach to the last session, session 0, or a named session
tmux a
tmux attach -t 0
tmux attach -t mysession
tmux a -t work

# Kill a session
tmux kill-session -t 0
```