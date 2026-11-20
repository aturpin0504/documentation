---
layout: default
title: Essential CLI Tools
parent: Quick Reference
grand_parent: RHEL
nav_order: 1
---

# Essential Command Line Tools

## Command Syntax Basics

| Task | Command |
|---|---|
| Basic structure | `command [options] [arguments]` |
| Example command | `ls -l /etc` |
| Locate a command or identify its type | `command -v ls` |
| Display detailed command type information | `type ls` |

## Getting Help

| Task | Command |
|---|---|
| Display quick command help | `command --help` |
| Display manual page | `man command` |
| Search within a manual page | <code>man passwd &#124; grep shell</code> |
| Display GNU info documentation | `info command` |

## Man Page Navigation

| Key | Action |
|---|---|
| `Space` / `Page Down` | Page down |
| `b` / `Page Up` | Page up |
| `g` | Go to beginning |
| `G` | Go to end |
| `/pattern` | Search forward |
| `n` | Next match |
| `N` | Previous match |
| `q` | Quit |

## Command History

| Task | Command |
|---|---|
| Show command history | `history` |
| Search history with grep | <code>history &#124; grep cat</code> |
| Execute command by number | `!number` |
| Clear in-memory history | `history -c` |
| Write history to file | `history -w` |
| Clear history and file | `history -c; history -w` |

## I/O Redirection

| Redirector | Description |
|---|---|
| `>` or `1>` | Redirect STDOUT; overwrite destination file |
| `>>` | Redirect STDOUT; append to destination file |
| `2>` | Redirect STDERR |
| `2>&1` | Redirect STDERR to STDOUT |
| `<` | Redirect STDIN |

| Task | Command |
|---|---|
| Redirect output to a file; overwrite existing content | `ls > output.txt` |
| Append output to a file | `ls >> output.txt` |
| Redirect errors to the null device | `ls ilwehgi 2> /dev/null` |
| Redirect both output and errors to a file | `ls whuhiu > errout 2>&1` |
| Redirect errors to the null device and output to a file | `ls ilwehgi Documents > output 2> /dev/null` |

## Pipes

| Task | Command |
|---|---|
| Pipe `ls` output to the `less` pager | <code>ls &#124; less</code> |
| Search running processes | <code>ps aux &#124; grep process</code> |
| Open a file with the `less` pager | `less /etc/passwd` |

## Environment Variables & Shell Configuration

### Displaying Environment Variables

| Task | Command |
|---|---|
| List all exported environment variables | `printenv` |
| Print a specific exported variable | `printenv PATH` |
| Print a shell variable through shell expansion | `echo "$PATH"` |

> `printenv` displays exported environment variables. Use `echo "$VARIABLE"` when you specifically want to view the value as expanded by the current shell.

### Setting a Global Environment Variable

Create a `.sh` drop-in file under `/etc/profile.d/` instead of editing `/etc/profile` directly, so operating system updates do not overwrite local configuration.

| Task | Value or file content |
|---|---|
| Create the configuration file | `/etc/profile.d/custom_env.sh` |
| Add this line to the file | `export MY_GLOBAL_VAR="value"` |
| Recommended owner | `root:root` |
| Recommended mode | `0644` |

On systems with a strict `umask`, confirm that the file remains readable by regular users.

```bash
chown root:root /etc/profile.d/custom_env.sh
chmod 0644 /etc/profile.d/custom_env.sh
```

### Shell Configuration Files

- `/etc/profile` — System-wide login-shell configuration
- `/etc/bashrc` — System-wide Bash configuration
- `~/.bash_profile` — Per-user login-shell configuration
- `~/.bashrc` — Per-user Bash configuration
- `~/.bash_history` — Per-user command history
- `~/.bash_logout` — Per-user logout actions

## Console Commands

| Task | Command |
|---|---|
| Show current username | `whoami` |
| Show logged-in users or sessions | `who` |
| Show logged-in users and activity | `w` — use `w -u` after `sudo` or `su` |
| Show your user, primary group, and supplementary groups | `id` |
| Show a specific user's identity and groups | `id username` |
| Show only your group names | `id -Gn` |
| Show only a specific user's group names | `id -Gn username` |

## Root Access Methods

| Method | Authentication | Description |
|---|---|---|
| `su` | Target account password | Switch to root in a subshell |
| `su -` | Root password | Switch to root with a full login environment |
| `su - username` | Target user's password | Switch to a user with a full login environment |
| `sudo` | Per sudo policy | Execute a command as root by default |
| `sudo -i` | Per sudo policy | Start an interactive root login shell |

### Creating and Modifying `sudoers` Files

Always edit `sudoers` with `visudo`. It validates syntax before saving and helps prevent an invalid policy from locking administrators out of `sudo`.

| Task | Command |
|---|---|
| Edit the main sudoers file | `visudo` |
| Create or edit a sudoers drop-in file | `visudo -f /etc/sudoers.d/custom_admins` |
| Validate the sudoers policy | `visudo -c` |

{: .warning }
> **Naming and Permissions for `/etc/sudoers.d/`**
>
> - Do not use periods (`.`) in a sudoers drop-in filename. Files with periods or a trailing `~` are ignored by the included-directory processing.
> - Use descriptive names with underscores or hyphens, such as `custom_admins` or `50_custom_admins`.
> - Ensure manually created or copied files are owned by `root:root` and have mode `0440`.

```bash
chown root:root /etc/sudoers.d/custom_admins
chmod 0440 /etc/sudoers.d/custom_admins
visudo -c
```

### Common `sudoers` Rule Formats: Allow All Commands

| Principal type | Rule format |
|---|---|
| Local user | `username ALL=(ALL) ALL` |
| Local group | `%groupname ALL=(ALL) ALL` |
| Active Directory user | `"aduser@domain.com" ALL=(ALL) ALL` |
| Active Directory group | `"%AD Group Name" ALL=(ALL) ALL` |

## Power Management

| Task | Command |
|---|---|
| Reboot the system immediately | `systemctl reboot` |
| Power off the system immediately | `systemctl poweroff` |
| Halt the OS without powering off the machine (uncommon) | `systemctl halt` |
| Schedule a reboot — template | `shutdown -r +<minutes> "message"` |
| Schedule a reboot — example | `shutdown -r +15 "Scheduled maintenance reboot in 15 minutes"` |
| Cancel a scheduled shutdown or reboot | `shutdown -c` |