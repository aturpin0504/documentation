---
layout: default
title: Essential CLI Tools
parent: Quick Reference
grand_parent: RHEL
---

# Essential Command Line Tools

## Command Syntax Basics

| Task | Command |
|---|---|
| Basic structure | `command [options] [arguments]` |
| Example command | `ls -l /etc` |
| Show path to executable | `which time` |
| Display command type | `type time` |

## Getting Help

| Task | Command |
|---|---|
| Display quick command help | `command --help` |
| Display manual page | `man command` |
| Search within a manual page | <code>man passwd &#124; grep shell</code> |
| Display info page | `pinfo command` |

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
| `>` or `1>` | Redirect STDOUT (overwrites) |
| `>>` | Redirect STDOUT (appends) |
| `2>` | Redirect STDERR |
| `2>&1` | Redirect STDERR to STDOUT |
| `<` | Redirect STDIN |

| Task | Command |
|---|---|
| Redirect output to file (overwrites) | `ls > output.txt` |
| Append output to file | `ls >> output.txt` |
| Redirect errors to null device | `ls ilwehgi 2> /dev/null` |
| Redirect both output and errors | `ls whuhiu > errout 2>&1` |
| Redirect errors to null, output to file | `ls ilwehgi Documents 2> /dev/null > output` |

## Pipes

| Task | Command |
|---|---|
| Pipe `ls` output to `less` pager | <code>ls &#124; less</code> |
| Search running processes | <code>ps aux &#124; grep process</code> |
| Open file with `less` pager | `less /etc/passwd` |

## Environment Variables & Shell Configuration

### Displaying Environment Variables

```bash
env                      # List all exported environment variables
printenv                 # Same as env
printenv PATH            # Print a specific variable
echo $PATH               # Print a specific variable via shell expansion
```

### Setting a Global Environment Variable

Create a `.sh` drop-in file under `/etc/profile.d/` instead of editing `/etc/profile` directly, so OS updates do not overwrite your changes:

```bash
# /etc/profile.d/custom_env.sh
export MY_GLOBAL_VAR="value"
```

Ensure the file is owned by `root:root` and is world-readable (`0644`), especially on STIG systems with a strict `umask` (`027` or `077`) where new files could otherwise be unreadable by regular users:

```bash
chown root:root /etc/profile.d/custom_env.sh
chmod 0644 /etc/profile.d/custom_env.sh
```

> **Shell Configuration Files**
>
> - `/etc/profile` — System-wide login shell
> - `/etc/bashrc` — System-wide shell settings
> - `~/.bash_profile` — User login shell
> - `~/.bashrc` — User shell settings
> - `~/.bash_history` — User command history
> - `~/.bash_logout` — User logout actions

## Console Commands

| Task | Command |
|---|---|
| Show current username | `whoami` |
| Show logged-in users/sessions | `who` |
| Show logged-in users and activity | `w` — use `w -u` after `sudo` or `su` |
| Show your user, primary group, and supplementary groups | `id` |
| Show a specific user's identity and groups | `id username` |
| Show only your group names | `id -Gn` |
| Show only a specific user's group names | `id -Gn username` |

## Root Access Methods

| Method | Password | Description |
|---|---|---|
| `su` | Root | Switch to root (subshell) |
| `su -` | Root | Root with full login environment |
| `su - username` | Target user's | Switch to user with full environment |
| `sudo` | Your password | Execute command as root |
| `sudo -i` | Your password | Interactive root shell |

## Creating & Modifying `sudoers` Files

Always edit `sudoers` with `visudo`. It validates syntax before saving, helping prevent a typo from locking you out of `sudo`. On RHEL, `visudo` uses `vi` by default.

```bash
visudo                                   # Edit the default /etc/sudoers file
visudo -f /etc/sudoers.d/custom_admins   # Create or edit a custom drop-in file (recommended)
```

> [!WARNING]
> ### Naming and Permissions for `/etc/sudoers.d/`
>
> - **No periods:** `/etc/sudoers.d/` ignores filenames containing a period (`.`) or ending in `~`. Use underscores instead—for example, `custom_admins`, not `custom.admins`.
> - **Permissions (`0440`):** Files must be owned by `root:root` and have mode `0440` (`-r--r-----`), or `sudo` will ignore them. `visudo -f` sets this automatically; if creating or copying a file manually, run:
>
> ```bash
> chown root:root /etc/sudoers.d/custom_admins
> chmod 0440 /etc/sudoers.d/custom_admins
> ```

### Common `sudoers` Rule Formats: Allow All Commands

```sudoers
# Local user
username ALL=(ALL) ALL

# Local group
%groupname ALL=(ALL) ALL

# Active Directory user (wrap in quotes)
"aduser@domain.com" ALL=(ALL) ALL

# Active Directory group (wrap in quotes, especially with spaces)
"%AD Group Name" ALL=(ALL) ALL
```

## Power Management

| Task | Command |
|---|---|
| Reboot the system immediately | `systemctl reboot` |
| Power off the system immediately | `systemctl poweroff` |
| Halt the OS without powering off the machine (uncommon) | `systemctl halt` |
| Schedule a reboot | `shutdown -r +<minutes> "message"` |
| Schedule a reboot | `shutdown -r +15 "Scheduled maintenance reboot in 15 minutes"` |
| Cancel a scheduled shutdown or reboot | `shutdown -c` |