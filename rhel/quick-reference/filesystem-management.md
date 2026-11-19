# File System Management

Daily file system hierarchy, mounting, navigation, permissions/SELinux context, searching, links, and archiving essentials for RHEL.

---

## RHEL File System Hierarchy

| Directory | Purpose |
| :--- | :--- |
| `/` | Root of the entire file system hierarchy. |
| `/boot` | Kernel (`vmlinuz`), `initramfs`, and GRUB bootloader files (`/boot/efi` for UEFI). |
| `/dev` | Device files (disks, terminals, `/dev/null`, `/dev/urandom`). |
| `/etc` | System-wide configuration files. |
| `/home` | Standard user home directories. |
| `/root` | Home directory for the `root` user. |
| `/media` | Temporary mount points for removable media. |
| `/mnt` | Temporary mount points for manually mounted file systems. |
| `/opt` | Optional/third-party add-on software packages. |
| `/proc` | Virtual file system exposing kernel and process state. |
| `/sys` | Virtual file system exposing hardware and kernel subsystem info. |
| `/run` | Runtime data since last boot (PID files, sockets; `tmpfs`). |
| `/srv` | Data for services provided by the system (e.g., web/FTP). |
| `/tmp` | Temporary files (often cleared on reboot or via `systemd-tmpfiles`). |
| `/usr` | User binaries (`/usr/bin`), admin binaries (`/usr/sbin`), libraries (`/usr/lib64`). (`/bin`, `/sbin`, `/lib`, `/lib64` are symlinks here.) |
| `/var` | Variable data: logs (`/var/log`), caches, spool files, databases, `/var/tmp`. |

---

## Mounting, Unmounting & `/etc/fstab`

```bash
lsblk -f                          # View block devices, file system types, UUIDs, and mount points
findmnt                           # Tree view of currently mounted file systems
mount /dev/sdb1 /mnt/data         # Temporarily mount a device
umount /mnt/data                  # Unmount by mount point (or device path)
```

### Applying `/etc/fstab` Changes

```bash
systemctl daemon-reload           # Required on RHEL 8/9+ so systemd picks up /etc/fstab edits
mount -a                          # Mount all entries in /etc/fstab (also validates syntax)
mount -o remount /var/tmp         # Remount an already-mounted file system with updated options
```

### STIG-Compliant Partitioning & Mount Options

DISA STIG for RHEL requires separate partitions/logical volumes for key directories and restrictive mount options (`nodev`, `nosuid`, `noexec`) to limit privilege escalation and binary execution:

| Mount Point | Required STIG Options | Notes |
| :--- | :--- | :--- |
| `/tmp` | `defaults,nodev,nosuid,noexec` | Prevents device files, SUID binaries, and execution from `/tmp`. |
| `/var/tmp` | `defaults,nodev,nosuid,noexec` | Same protections as `/tmp`. |
| `/dev/shm` | `defaults,nodev,nosuid,noexec` | Shared memory `tmpfs` mount. |
| `/var` | `defaults,nodev` | Separate partition required. |
| `/var/log` | `defaults,nodev,nosuid,noexec` | Separate partition for system logs. |
| `/var/log/audit` | `defaults,nodev,nosuid,noexec` | Separate partition for `auditd` logs. |
| `/home` | `defaults,nodev,nosuid` | Separate partition; `noexec` omitted if users run local scripts/binaries. |
| `/boot` | `defaults,nodev,nosuid` | Separate partition (and `nodev,nosuid` on `/boot/efi` where applicable). |

Example `/etc/fstab` entry:

```text
UUID=1234-5678  /tmp  xfs  defaults,nodev,nosuid,noexec  0 0
```

---

## Inspecting Files & Directories

```bash
pwd                      # Print current working directory
stat /etc/hosts          # Detailed metadata: size, inode, links, permissions, UID/GID, SELinux context, timestamps
file /usr/bin/passwd     # Identify file type by content (magic bytes), not extension
```

### Navigating with `cd`

```bash
cd                       # Go to your home directory ($HOME)
cd ..                    # Go up one directory (parent directory)
cd -                     # Toggle back to the previous working directory
cd /var/log              # Go to a specific directory
```

### Listing Files with `ls`

```bash
ls -l                    # Long listing (permissions, link count, owner, group, size, mtime)
ls -a                    # Include hidden files (dotfiles)
ls -la                   # Long listing including hidden files
ls -lh                   # Human-readable file sizes (KB, MB, GB)
ls -ltr                  # Sort by modification time, newest at the bottom
ls -d /etc/              # List the directory entry itself rather than its contents
ls -Z                    # Show SELinux security context (user:role:type:level)
ls -laZ                  # Combine long listing, hidden files, and SELinux context
```

---

## Creating, Copying, Moving & Removing

### Creating & Removing Directories/Files

```bash
mkdir newdir             # Create a directory
mkdir -p a/b/c           # Create nested parent directories as needed (no error if they exist)

rm file.txt              # Remove a file
rm -f file.txt           # Force remove without prompt
rm -r mydir              # Recursively remove a directory and its contents
rm -rf mydir             # Force recursive removal (use with care)
```

### Copying, Moving, Renaming & SELinux Context

```bash
cp src.txt dst.txt       # Copy a file
cp -r srcdir dstdir      # Copy a directory recursively
cp -a srcdir dstdir      # Archive copy (preserves permissions, timestamps, symlinks, and SELinux context)
mv old.txt new.txt       # Move or rename a file/directory
```

> [!TIP]
> **Automatically Setting Destination SELinux Context on `cp` / `mv`**
> - **`cp`** creates a new file at the destination, so by default it **inherits the destination directory's SELinux context** (unless you pass `-a` or `--preserve=context`).
> - **`mv`** **preserves the source file's SELinux context** by default—which often breaks services when moving files from `/tmp` or `~` into `/var/www/html` or `/etc`.
> - Use **`mv -Z`** (or `cp -Z`) to automatically apply the destination's default SELinux context during the move/copy: `mv -Z ~/index.html /var/www/html/`
> - If a file already has the wrong context, fix it with `restorecon -v /path/to/file`.

---

## Searching with `find`

Always **quote wildcards** (e.g., `"*.log"`) so the shell doesn't expand them before `find` runs.

```bash
find /var/log -name "*.log"              # Find files matching a wildcard (case-sensitive)
find /etc -iname "*sssd*"                # Case-insensitive name search
find /var -type f -size +100M            # Find regular files larger than 100 MB
find /etc -type f -mtime -7              # Modified within the last 7 days (+7 = older than 7 days)
find /home -user jdoe                    # Owned by a specific user
find / -type f -perm /4000 2>/dev/null   # Find SUID binaries, discarding permission-denied errors
find /tmp -type f -mtime +30 -delete     # Delete matching files older than 30 days
find /var/log -name "*.log" -exec ls -lh {} +   # Run a command on all matches
```

---

## Hard Links vs. Symbolic Links

```bash
ln /path/to/target /path/to/hardlink     # Create a hard link
ln -s /path/to/target /path/to/symlink   # Create a symbolic (soft) link
```

| Feature | Hard Link (`ln`) | Symbolic Link (`ln -s`) |
| :--- | :--- | :--- |
| **Points to** | Same **inode** (actual data on disk) | File **path** (name) |
| **Cross filesystems?** | No (must be on the same filesystem) | Yes |
| **Link directories?** | No | Yes |
| **If original is deleted** | Data remains intact via the hard link | Link breaks ("dangling" symlink) |
| **When to use** | Keeping a file accessible in two locations on the same filesystem without duplicating space | General admin use: linking directories, pointing to versioned paths (`/opt/app -> /opt/app-2.1`), cross-filesystem links |

> [!WARNING]
> **Gotchas When Working with & Removing Links**
> - **Use full paths for symlinks:** Relative targets in `ln -s` resolve relative to the *symlink's directory*, not your current working directory. Using absolute paths avoids broken links.
> - **Never add a trailing slash when removing a symlink to a directory:** Use `rm mylink` or `unlink mylink`. Never run `rm -rf mylink/`—the trailing slash dereferences into the target directory!
> - **Hard link disk space:** Deleting one hard link only decrements the link count (column 2 in `ls -l`). Space is freed only when the link count hits `0` and no process holds the file open.

---

## Archiving & Compression (`tar`, `gzip`, `gunzip`)

### Working with `tar` + `gzip` (`.tar.gz` / `.tgz`)

```bash
tar -czvf backup.tar.gz /etc/sssd        # Create (-c) a gzipped (-z) archive, verbose (-v), filename (-f)
tar -tzvf backup.tar.gz                  # List (-t) contents without extracting
tar -xzvf backup.tar.gz                  # Extract (-x) into the current directory
tar -xzvf backup.tar.gz -C /tmp/restore  # Extract into a specific directory (-C)
```

*(Add `--selinux` or `--xattrs` when creating/extracting archives if you need to preserve SELinux contexts.)*

### Single-File Compression (`gzip` / `gunzip`)

```bash
gzip app.log          # Compresses app.log -> app.log.gz (replaces original)
gzip -k app.log       # Keep (-k) the original file alongside app.log.gz
gunzip app.log.gz     # Decompresses app.log.gz -> app.log (same as: gzip -d app.log.gz)
```