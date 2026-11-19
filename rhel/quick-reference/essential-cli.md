# Essential Command Line Tools

Daily command-line essentials for RHEL administration.

---

## Finding Help & Command Lookup

| Command | Purpose |
| :--- | :--- |
| `which <cmd>` | Shows the full path of an executable found in `$PATH`. |
| `type <cmd>` | Identifies whether `<cmd>` is a shell builtin, alias, function, or file path. |
| `<cmd> --help` | Prints a brief usage summary and available flags. |
| `man <cmd>` | Opens the full manual page for `<cmd>`. |

### Man Page Navigation

`man` uses `less` as its default pager:

| Key | Action |
| :--- | :--- |
| `Space` / `f` | Page forward |
| `b` | Page backward |
| `j` / `k` (or `↓` / `↑`) | Scroll down / up one line |
| `g` / `G` | Jump to top / bottom |
| `/<pattern>` | Search forward (`n` next match, `N` previous match) |
| `?<pattern>` | Search backward |
| `q` | Quit |

---

## Command History

```bash
history                  # List previously run commands with line numbers
!<number>                # Re-run the command at <number> from history
history -c; history -w   # Clear in-memory history (-c) and overwrite ~/.bash_history (-w)
```

---

## I/O Redirection

Linux processes use three standard streams: `0` (`stdin`), `1` (`stdout`), and `2` (`stderr`).

| Task | Syntax |
| :--- | :--- |
| Redirect `stdout` to a file (overwrite) | `command > file.txt` |
| Append `stdout` to a file | `command >> file.txt` |
| Discard errors (`stderr` to null device) | `command 2> /dev/null` |
| Send `stdout` and `stderr` to separate files | `command > out.txt 2> err.txt` |
| Send both `stdout` and `stderr` to the same file | `command &> both.txt` (or `command > both.txt 2>&1`) |
| Append both `stdout` and `stderr` to the same file | `command &>> both.txt` (or `command >> both.txt 2>&1`) |

---

## Paging Output with `less`

```bash
less /var/log/messages   # Page through a file
dmesg | less             # Pipe command output into less
command 2>&1 | less      # Include stderr when piping to less
```

Navigation inside `less` (same keys as `man`, plus `F` for live log following):

| Key | Action |
| :--- | :--- |
| `Space` / `f` | Page forward |
| `b` | Page backward |
| `j` / `k` (or `↓` / `↑`) | Scroll down / up one line |
| `g` / `G` | Jump to top / bottom |
| `/<pattern>` | Search forward (`n` next match, `N` previous match) |
| `?<pattern>` | Search backward |
| `F` | Follow file in real time (like `tail -f`; press `Ctrl+C` to stop following) |
| `q` | Quit |

---

## Environment Variables & Shell Configuration

### Displaying Environment Variables

```bash
env                      # List all exported environment variables
printenv                 # Same as env
printenv PATH            # Print a specific variable
echo $PATH               # Print a specific variable via shell expansion
```

### Setting a Global Environment Variable

Create a `.sh` drop-in file under `/etc/profile.d/` instead of editing `/etc/profile` directly so OS updates don't overwrite your changes:

```bash
# /etc/profile.d/custom_env.sh
export MY_GLOBAL_VAR="value"
```

Ensure the file is owned by `root:root` and world-readable (`0644`), especially on STIG systems with a strict `umask` (`027` or `077`) where new files would otherwise be unreadable by regular users:

```bash
chmod 0644 /etc/profile.d/custom_env.sh
```

### Shell Configuration Files

| Scope | File | When It Runs |
| :--- | :--- | :--- |
| **System-wide** | `/etc/profile` | Login shells for all users (also sources `/etc/profile.d/*.sh`). |
| **System-wide** | `/etc/bashrc` | Interactive non-login shells for all users. |
| **Per-user** | `~/.bash_profile` | User login shells (typically sources `~/.bashrc`). |
| **Per-user** | `~/.bashrc` | User interactive non-login shells (aliases, functions, prompt). |
| **Per-user** | `~/.bash_logout` | Executed when a user exits a login shell. |

---

## Users, Sessions & Identity

```bash
who                      # Show currently logged-in users, TTY, login time, and host
id                       # Show current user's UID, primary group (gid), and all group memberships
id <username>            # Same output for a specific user (includes primary group + all groups)
```

---

## Root Access Methods

| Command | Behavior | Password Required |
| :--- | :--- | :--- |
| `su` | Switches to `root`, keeping your current environment and directory. | **`root` password** |
| `su -` | Switches to `root` with a full login shell (loads root's `$PATH` and `/root`). | **`root` password** |
| `su - <username>` | Switches to `<username>` with a full login shell. | **`<username>`'s password** (none if run by `root`) |
| `sudo <command>` | Runs a single command as `root`. | **Your own password** |
| `sudo -i` | Opens an interactive `root` login shell (cleaner alternative to `sudo su`). | **Your own password** |

---

## Creating & Modifying `sudoers` Files

Always edit `sudoers` with `visudo`—it validates syntax before saving so a typo won't lock you out of `sudo`. On RHEL, `visudo` uses `vi` by default.

```bash
visudo                                   # Edit the default /etc/sudoers file
visudo -f /etc/sudoers.d/custom_admins   # Create/edit a custom drop-in file (recommended)
```

> [!WARNING]
> **Naming & Permissions for `/etc/sudoers.d/`**
> - **No periods:** `/etc/sudoers.d/` ignores filenames containing a period (`.`) or ending in `~`. Use underscores instead (e.g., `custom_admins`, not `custom.admins`).
> - **Permissions (`0440`):** Files must be owned by `root:root` with mode `0440` (`-r--r-----`), or `sudo` will ignore them. `visudo -f` sets this automatically; if creating/copying manually, run `chown root:root` and `chmod 0440`.

### Common `sudoers` Rule Formats (Allow All Commands)

```text
# Local user
username ALL=(ALL) ALL

# Local group
%groupname ALL=(ALL) ALL

# Active Directory user (wrap in quotes)
"aduser@domain.com" ALL=(ALL) ALL

# Active Directory group (wrap in quotes, especially with spaces)
"%AD Group Name" ALL=(ALL) ALL
```

---

## Power Management

```bash
poweroff                 # Shut down and power off the system (or: systemctl poweroff)
reboot                   # Shut down and restart the system (or: systemctl reboot)
```