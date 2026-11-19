---
layout: default
title: File System Management
parent: Quick Reference
grand_parent: RHEL
---

# File System Management

Daily file system hierarchy, mounting, navigation, permissions and SELinux context, searching, links, archiving, and compression essentials for RHEL.

## RHEL File System Hierarchy

| Directory | Purpose |
|---|---|
| `/` | Root of the entire file system hierarchy. |
| `/boot` | Kernel (`vmlinuz`), `initramfs`, and GRUB bootloader files. On UEFI systems, EFI files are commonly under `/boot/efi`. |
| `/dev` | Device files, including disks, terminals, `/dev/null`, and `/dev/urandom`. |
| `/etc` | System-wide configuration files. |
| `/home` | Standard user home directories. |
| `/root` | Home directory for the `root` user. |
| `/media` | Temporary mount points for removable media. |
| `/mnt` | Temporary mount points for manually mounted file systems. |
| `/opt` | Optional or third-party add-on software packages. |
| `/proc` | Virtual file system exposing kernel and process state. |
| `/run` | Runtime data since the last boot, including PID files and sockets; typically a `tmpfs`. |
| `/srv` | Data for services provided by the system, such as web or FTP services. |
| `/sys` | Virtual file system exposing hardware and kernel-subsystem information. |
| `/tmp` | Temporary files; often cleared on reboot or by `systemd-tmpfiles`. |
| `/usr` | User binaries (`/usr/bin`), administrative binaries (`/usr/sbin`), libraries (`/usr/lib64`), and related program files. On modern RHEL, `/bin`, `/sbin`, `/lib`, and `/lib64` are commonly symbolic links into `/usr`. |
| `/var` | Variable data, including logs (`/var/log`), caches, spool files, databases, and `/var/tmp`. |

## Mounting, Unmounting, and `/etc/fstab`

```bash
lsblk -f                          # View block devices, file system types, UUIDs, and mount points
mount                             # Show all mounted file systems
findmnt                           # Tree view of currently mounted file systems
findmnt /home                     # Show a specific mount point
mount /dev/sdb1 /mnt/data         # Temporarily mount a device
umount /mnt/data                  # Unmount by mount point or device path
df -h                             # Display disk space in human-readable form
df -Th                            # Display disk space and file system types
```

### Applying `/etc/fstab` Changes

```bash
systemctl daemon-reload           # On RHEL 8/9+, reload systemd after /etc/fstab edits
mount -a                          # Mount all /etc/fstab entries and validate syntax
mount -o remount /var/tmp         # Remount an existing file system with updated options
```

### STIG-Compliant Partitioning and Mount Options

DISA STIG guidance for RHEL may require separate partitions or logical volumes for key directories and restrictive mount options such as `nodev`, `nosuid`, and `noexec` to reduce privilege-escalation and binary-execution risk.

| Mount Point | Required STIG Options | Notes |
|---|---|---|
| `/boot` | `defaults,nodev,nosuid` | Separate partition. |
| `/boot/efi` | `umask=0077,shortname=winnt,nosuid,nodev` | Separate partition; where applicable. |
| `/home` | `defaults,nodev,noexec,nosuid` | Separate partition for user home directories. |
| `/tmp` | `defaults,nodev,noexec,nosuid` | Prevents device files, SUID binaries, and execution from `/tmp`. |
| `/var` | `defaults,nodev` | Separate partition required. |
| `/var/log` | `defaults,nodev,noexec,nosuid` | Separate partition for system logs. |
| `/var/log/audit` | `defaults,nodev,noexec,nosuid` | Separate partition for `auditd` logs. |
| `/var/tmp` | `defaults,nodev,noexec,nosuid` | Same protections as `/tmp`. |
| `/dev/shm` | `defaults,relatime,inode64,nodev,noexec,nosuid` | Shared-memory `tmpfs` mount. |

Example `/etc/fstab` entry:

```fstab
UUID=1234-5678  /tmp  xfs  defaults,nodev,nosuid,noexec  0 0
```

## Inspecting Files and Directories

```bash
pwd                      # Print the current working directory
stat /etc/hosts          # Detailed metadata: inode, links, permissions, UID/GID, SELinux context, and timestamps
file /usr/bin/passwd     # Identify file type by content rather than file extension
```

## Directory Navigation

| Task | Command |
|---|---|
| Print working directory | `pwd` |
| Change to home directory | `cd` |
| Change to a specific directory | `cd /etc` |
| Change to parent directory | `cd ..` |
| Change to the previous directory | `cd -` |

Examples:

```bash
cd                       # Go to your home directory ($HOME)
cd /var/log              # Go to a specific directory
cd ..                    # Go up one directory
cd -                     # Toggle to the previous working directory
```

## Directory Management

| Task | Command |
|---|---|
| Create a directory | `mkdir newdir` |
| Create nested parent directories | `mkdir -p parent/child/grandchild` |
| Remove a directory and its contents | `rm -rf directory` |

```bash
mkdir newdir             # Create a directory
mkdir -p a/b/c           # Create nested directories as needed; does not fail if they already exist

rm -r mydir              # Recursively remove a directory and its contents
rm -rf mydir             # Force recursive removal; use with care
```

## Wildcards

| Wildcard | Description |
|---|---|
| `*` | Matches any number of characters. |
| `?` | Matches exactly one character. |
| `[auo]` | Matches one character from the specified set. |
| `[a-e]` | Matches one character from the specified range. |

## Listing Files with `ls`

| Task | Command |
|---|---|
| List directory contents | `ls` |
| Long listing | `ls -l` |
| Show hidden files | `ls -a` |
| Long listing including hidden files | `ls -la` |
| Human-readable sizes | `ls -lh` |
| Sort by time, newest first | `ls -lt` |
| Sort by time, oldest first | `ls -lrt` |
| Sort by size, largest first | `ls -lS` |
| Recursive listing | `ls -R` |
| List the directory entry itself | `ls -d /etc/` |
| Show SELinux context | `ls -Z` |
| Long listing, hidden files, and SELinux context | `ls -laZ` |

## Creating, Copying, Moving, and Removing Files

### File Management Commands

| Task | Command |
|---|---|
| Create an empty file or update its timestamp | `touch newfile` |
| Copy a file | `cp file1 file2` |
| Copy a directory recursively | `cp -r dir1 dir2` |
| Archive copy and preserve attributes | `cp -a source dest` |
| Copy verbosely | `cp -v file1 file2` |
| Copy interactively with prompts | `cp -i file1 file2` |
| Rename a file | `mv oldname newname` |
| Move a file | `mv file1 /tmp/` |
| Remove a file | `rm file1` |
| Force-remove a file | `rm -f file1` |

Examples:

```bash
touch newfile            # Create an empty file or update timestamp

cp src.txt dst.txt       # Copy a file
cp -r srcdir dstdir      # Copy a directory recursively
cp -a srcdir dstdir      # Preserve permissions, timestamps, symlinks, and SELinux context
cp -v file1 file2        # Verbose copy
cp -i file1 file2        # Prompt before overwriting

mv old.txt new.txt       # Rename or move a file/directory
mv file1 /tmp/           # Move a file to /tmp

rm file.txt              # Remove a file
rm -f file.txt           # Remove without prompting
```

> [!TIP]
> ### SELinux Context When Using `cp` and `mv`
>
> - `cp` creates a new destination file, which normally receives the destination directory's SELinux context unless `-a` or `--preserve=context` is used.
> - `mv` normally preserves the source file's SELinux context. This can cause problems when moving content from `/tmp` or a user home directory into locations such as `/var/www/html` or `/etc`.
> - Use `mv -Z` or `cp -Z` to apply the destination's default SELinux context during the operation:
>
> ```bash
> mv -Z ~/index.html /var/www/html/
> ```
>
> - If a file already has an incorrect context, restore the expected context:
>
> ```bash
> restorecon -v /path/to/file
> ```

## File Search with `find`

Always quote wildcards, such as `"*.log"`, so the shell does not expand them before `find` runs.

| Task | Command |
|---|---|
| Find files by name | `find / -name "*.conf"` |
| Find files by name, case-insensitive | `find / -iname "*.CONF"` |
| Find files owned by a user | `find / -user username` |
| Find files owned by a group | `find / -group groupname` |
| Find files by UID | `find / -uid 1000` |
| Find files by GID | `find / -gid 1000` |
| Find files larger than 100 MB | `find / -size +100M` |
| Find files smaller than 10 KB | `find / -size -10k` |
| Find files modified in the last 7 days | `find / -mtime -7` |
| Find files modified more than 30 days ago | `find / -mtime +30` |
| Find files accessed today | `find / -atime 0` |
| Find files by type: file or directory | `find / -type f` or `find / -type d` |
| Find by permissions | `find / -perm 0644` |
| Find SUID files | `find / -perm -4000` |
| Find SGID files | `find / -perm -2000` |
| Find world-writable files | `find / -perm -002` |
| Find and execute a command | `find / -name "*.log" -exec rm {} \;` |
| Find and request confirmation before execution | `find / -name "*.tmp" -ok rm {} \;` |

Examples:

```bash
find /var/log -name "*.log"              # Find matching files; case-sensitive
find /etc -iname "*sssd*"                # Case-insensitive name search
find /var -type f -size +100M            # Find regular files larger than 100 MB
find /etc -type f -mtime -7              # Files modified within 7 days
find /home -user jdoe                    # Files owned by a specific user
find / -type f -perm /4000 2>/dev/null   # Find SUID files and suppress permission errors
find /tmp -type f -mtime +30 -delete     # Delete matching files older than 30 days
find /var/log -name "*.log" -exec ls -lh {} +  # Run a command against all matches
```

## Hard Links and Symbolic Links

```bash
ln /path/to/target /path/to/hardlink     # Create a hard link
ln -s /path/to/target /path/to/symlink   # Create a symbolic (soft) link
rm link                                  # Remove a link
```

| Feature | Hard Link: `ln` | Symbolic Link: `ln -s` |
|---|---|---|
| Points to | The same inode and underlying file data. | A file-system path. |
| Cross file systems? | No; must remain on the same file system. | Yes. |
| Can link to directories? | No. | Yes. |
| If original is deleted | Data remains available through the hard link. | The link becomes broken or “dangling.” |
| Typical use | Make one file available at multiple paths without duplicating disk space. | General administrative linking, directory links, versioned paths, and cross-file-system links. |

### Hard Link Restrictions

- You must own the file.
- Hard links cannot cross devices or partitions.
- Hard links cannot point to directories.

### Symbolic Links

Symbolic links can cross devices and can point to directories.

> [!WARNING]
> ### Link Management Gotchas
>
> - **Use full paths for symbolic links:** Relative targets in `ln -s` are evaluated relative to the symbolic link’s directory—not necessarily your current directory. Absolute paths help avoid broken links.
> - **Do not use a trailing slash when removing a symbolic link to a directory:** Use `rm mylink` or `unlink mylink`. Do **not** run `rm -rf mylink/`; the trailing slash can dereference the link and affect the target directory.
> - **Hard-link disk space:** Deleting one hard link only decreases the link count shown in the second column of `ls -l`. Disk space is released only when the link count reaches zero and no process still has the file open.

## Archiving with `tar`

### Common Archive Operations

| Task | Command |
|---|---|
| Create an archive | `tar -cvf archive.tar /etc` |
| Extract an archive | `tar -xvf archive.tar` |
| Extract to a directory | `tar -xvf archive.tar -C /tmp` |
| List archive contents | `tar -tvf archive.tar` |
| Append a file to an archive | `tar -rvf archive.tar newfile` |
| Update archive with newer files | `tar -uvf archive.tar /home` |

### `tar` Options

| Option | Description |
|---|---|
| `c` | Create an archive. |
| `x` | Extract an archive. |
| `t` | List archive contents. |
| `v` | Verbose output. |
| `f` | Specify archive filename. |
| `z` | Use gzip compression. |
| `j` | Use bzip2 compression. |
| `J` | Use xz compression. |
| `C` | Extract to a specified directory. |
| `r` | Append files to an archive. |
| `u` | Update an archive with newer files. |

### Compressed Archives

| Task | Command |
|---|---|
| Create gzip archive | `tar -czvf archive.tar.gz /etc` |
| Create bzip2 archive | `tar -cjvf archive.tar.bz2 /etc` |
| Create xz archive | `tar -cJvf archive.tar.xz /etc` |
| Extract gzip archive | `tar -xzvf archive.tar.gz` |

### Working with `tar` and gzip

```bash
tar -czvf backup.tar.gz /etc/sssd        # Create a gzipped archive
tar -tzvf backup.tar.gz                  # List contents without extracting
tar -xzvf backup.tar.gz                  # Extract into the current directory
tar -xzvf backup.tar.gz -C /tmp/restore  # Extract to a specified directory
```

> [!TIP]
> When archiving content that requires SELinux extended attributes, use the appropriate `tar` options, such as `--selinux` or `--xattrs`, when creating or extracting the archive.

## Compression Tools

| Task | Command |
|---|---|
| Compress with gzip | `gzip file.tar` |
| Compress while keeping the original file | `gzip -k file.tar` |
| Decompress gzip | `gunzip file.tar.gz` |
| Compress with bzip2 | `bzip2 file.tar` |
| Decompress bzip2 | `bunzip2 file.tar.bz2` |
| Compress with xz | `xz file.tar` |
| Decompress xz | `unxz file.tar.xz` |

### Single-File gzip Compression

```bash
gzip app.log          # Compress app.log to app.log.gz and replace the original
gzip -k app.log       # Keep app.log and create app.log.gz
gunzip app.log.gz     # Decompress app.log.gz to app.log
```

## Important Tips

- Use `cp -a` when you need to preserve permissions, timestamps, symbolic links, and SELinux context.
- Use `rm -rf` carefully; there is no undo operation.
- Before extracting an archive, consider using `tar -tvf archive.tar` to inspect its contents.
- Never add a trailing slash when removing a symbolic link to a directory.