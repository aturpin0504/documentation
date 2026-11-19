---
layout: default
title: Network Fundamentals
parent: Quick Reference
grand_parent: RHEL
---

# Network Fundamentals

## IP Address Types

- **IPv4:** 32-bit address format, such as `192.168.10.100`.
- **IPv6:** 128-bit address format, such as `fe80::...`.
- **Private IPv4 ranges:**
  - `10.0.0.0/8`
  - `172.16.0.0/12`
  - `192.168.0.0/16`

## Network Validation

| Task | Command |
|---|---|
| Show all IP addresses | `ip addr show` |
| Show a specific interface | `ip addr show dev ens33` |
| Show link state | `ip link show` |
| Show link statistics | `ip -s link show` |
| Show routing table | `ip route show` |
| Test connectivity | `ping -c 4 8.8.8.8` |
| Show listening and established sockets | `ss -tul` |

### `ss` Options

| Option | Description |
|---|---|
| `-t` | Show TCP sockets |
| `-u` | Show UDP sockets |
| `-l` | Show listening sockets |
| `-n` | Do not resolve service or host names |
| `-a` | Show all sockets |

> [!WARNING]
> Do not use `ifconfig` on RHEL 8. Use the `ip` command instead.

## NetworkManager

| Task | Command |
|---|---|
| Check NetworkManager status | `systemctl status NetworkManager` |
| Show current NetworkManager permissions | `nmcli gen permissions` |
| Open the NetworkManager text user interface | `nmtui` |
| Apply connection changes without restarting the interface | `nmcli device reapply ens192` |

> [!NOTE]
> Replace `ens192` with the applicable interface name.

### `nmtui` Options

- **Edit a Connection**
- **Activate a Connection**
- **Set System Hostname**

After modifying a connection in `nmtui`, deactivate and reactivate the connection to apply the changes.

## Network Configuration Files

| File | Purpose |
|---|---|
| `/etc/sysconfig/network-scripts/ifcfg-*` | Network connection configuration files. |
| `/etc/hostname` | System hostname. |
| `/etc/hosts` | Static hostname-to-IP mappings. |
| `/etc/resolv.conf` | DNS configuration; typically auto-generated. |
| `/etc/nsswitch.conf` | Name-resolution lookup order. |

## Hostname Configuration

| Task | Command |
|---|---|
| Set hostname using the preferred method | `hostnamectl set-hostname server1.example.com` |
| Show hostname information | `hostnamectl status` |
| Set hostname through NetworkManager text UI | `nmtui` |
| Edit hostname file directly | `vi /etc/hostname` |

## DNS Configuration

| Task | Command |
|---|---|
| View DNS configuration | `cat /etc/resolv.conf` |
| Edit static hostname mappings | `vi /etc/hosts` |
| Test name resolution through `/etc/hosts` and DNS | `getent hosts server1.example.com` |
| Perform a DNS lookup | `host server1.example.com` |

### `/etc/hosts` Format

```text
IP_ADDRESS FQDN ALIAS
```

Example:

```text
10.0.0.10 server1.example.com server1
```

> [!WARNING]
> Do not edit `/etc/resolv.conf` directly. NetworkManager can overwrite the file. Use `nmtui` or `nmcli` to manage DNS settings.

## Login Messages

| File | Display Time | Description |
|---|---|---|
| `/etc/motd` | After login | Message of the Day; displayed for shell logins. |
| `/etc/issue` | Before login | Console login message. |
| `/etc/issue.net` | Before login | SSH or network login message. |

## SSH Service Management

| Task | Command |
|---|---|
| Check SSH daemon status | `systemctl status sshd` |
| Start SSH daemon | `systemctl start sshd` |
| Enable SSH daemon at boot | `systemctl enable sshd` |
| Restart SSH daemon | `systemctl restart sshd` |

## SSH Configuration Files

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

# Lock an account; prevents account login
usermod -L username

# Remove or rename the authorized_keys file
rm ~/.ssh/authorized_keys
mv ~/.ssh/authorized_keys ~/.ssh/authorized_keys.disabled
```

You can also restrict SSH access through `/etc/ssh/sshd_config`:

```text
DenyUsers username
```

Or use an allowlist:

```text
AllowUsers user1 user2
```

After modifying `sshd_config`, validate and reload the SSH service:

```bash
sshd -t
systemctl reload sshd
```

## SSH Troubleshooting

| Task | Command |
|---|---|
| Verify that `sshd` is running | `systemctl status sshd` |
| Check whether port 22 is listening | `ss -tlnp \| grep :22` |
| Check firewall rules | `firewall-cmd --list-all` |
| Monitor SSH authentication attempts | `tail -f /var/log/secure` |
| Follow `sshd` journal logs | `journalctl -u sshd -f` |
| Show IPv4 addresses | `ip a \| grep 'inet '` |

## SFTP File Transfer

### SFTP Context

- **Local:** Your workstation.
- **Remote:** The SSH server to which you are connected.
- Commands beginning with `l` operate on the local system.

| Command | Description |
|---|---|
| `put file` | Upload a file to the remote system. |
| `get file` | Download a file from the remote system. |
| `get -r directory` | Download a directory recursively. |
| `ls` | List the remote directory. |
| `lls` | List the local directory. |
| `cd dir` | Change the remote directory. |
| `lcd dir` | Change the local directory. |
| `pwd` | Show the remote working directory. |
| `lpwd` | Show the local working directory. |
| `quit` | Exit the SFTP session. |

## `tmux`: Terminal Multiplexer

| Task | Command |
|---|---|
| Install `tmux` | `yum install -y tmux` |
| Start a new session | `tmux` |
| Start a named session | `tmux new -s mysession` |
| List all sessions | `tmux ls` |
| Attach to the last session | `tmux a` |
| Attach to session `0` | `tmux attach -t 0` |
| Attach to a named session | `tmux attach -t mysession` |
| Attach using shorthand | `tmux a -t work` |
| Kill a session | `tmux kill-session -t 0` |
| Detach from a session | `Ctrl-b d` |

### `tmux` Sessions

- Sessions created without `-s` use numeric names such as `0`, `1`, and `2`.
- Sessions created with `-s` use custom names.
- Most `tmux` keyboard commands begin with the `Ctrl-b` prefix.

## Important Tips

- Use `nmtui` for straightforward network configuration changes.
- Use the `ip` command rather than `ifconfig` on RHEL 8.
- Use `hostnamectl` for hostname changes.
- Do not edit `/etc/resolv.conf` directly when it is managed by NetworkManager.
- Use `/etc/hosts` for static hostname mappings.
- Use `tmux` to maintain persistent terminal sessions during SSH connections.