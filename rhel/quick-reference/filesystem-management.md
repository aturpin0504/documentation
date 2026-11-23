---
layout: default
title: File System Management
parent: Quick Reference
grand_parent: RHEL
nav_order: 2
---

# File System Management

File system hierarchy, mounting, navigation, permissions, SELinux context, searching, links, archiving, and compression essentials for RHEL.

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

## Disk Usage

Use `df` to view free space by mounted filesystem. Use `du` to identify which directories or files consume disk space.

```bash
# Show disk-space usage for mounted filesystems
df -h

# Show total size of the current directory
du -sh .

# Show size of a specific directory
du -sh /var/log

# Show sizes of immediate subdirectories in the current directory
du -sh */

# Show first-level directories under /var, sorted from smallest to largest
du -sh /var/* | sort -h

# Show the 10 largest entries under /var
du -ah /var | sort -h | tail -n 10

# Stay on the current filesystem; do not cross into mounted filesystems
du -xsh /var
```

{: .important }
> Use `df -h` to find a full filesystem, then use `du -xsh /path` and `du -sh /path/*` to identify the directories consuming space on that filesystem.

## Mounting, Unmounting, and `/etc/fstab`

A **mount point** is a directory where Linux attaches another file system to the existing directory tree.

For example, before mounting a device, `/mnt/data` is an ordinary directory that is part of the root (`/`) file system. After mounting `/dev/sdb1` on `/mnt/data`, the directory becomes the access point for the file system on `/dev/sdb1`.

| Task | Command or configuration |
|---|---|
| View block devices, file system types, UUIDs, and mount points | `lsblk -f` |
| List mounted file systems in a tree view | `findmnt` |
| Show the file system containing a path | `findmnt -T /home` |
| Verify that a directory is an exact mount point | `findmnt -M /mnt/data` |
| Create a mount-point directory if needed | `mkdir -p /mnt/data` |
| Temporarily mount a device | `mount /dev/sdb1 /mnt/data` |
| Unmount by mount point or device path | `umount /mnt/data` |
| Display disk space in human-readable form | `df -h` |
| Display disk space and file system types | `df -Th` |
| Define a persistent mount | Add a `UUID=<uuid> /mnt/data <fstype> defaults 0 0` entry to `/etc/fstab` |
| Reload generated systemd mount units after editing `/etc/fstab` | `systemctl daemon-reload` |
| Attempt to mount applicable `/etc/fstab` entries | `mount -a` |

### Understanding `findmnt -T` and `findmnt -M`

| Question | Command | Meaning |
|---|---|---|
| Which file system contains this path? | `findmnt -T /mnt/data` | Displays the file system that contains `/mnt/data`. If `/mnt/data` is an ordinary directory, this might show the root (`/`) file system. |
| Is this exact directory a mount point? | `findmnt -M /mnt/data` | Displays information only if `/mnt/data` itself is a mount point. |
| What is mounted on this directory? | `findmnt -M /mnt/data -o TARGET,SOURCE,FSTYPE,OPTIONS` | Displays the mount point, source device, file system type, and mount options. |

For example:

| Condition | `findmnt -T /mnt/data` | `findmnt -M /mnt/data` |
|---|---|---|
| `/mnt/data` is an ordinary directory under `/` | Shows the root (`/`) file system | Shows no result |
| `/dev/sdb1` is mounted on `/mnt/data` | Shows `/dev/sdb1` mounted on `/mnt/data` | Shows `/dev/sdb1` mounted on `/mnt/data` |

{: .warning }
> **Verify the mount point before mounting**
>
> Use `findmnt -M /mnt/data` before mounting a device.
>
> If the command returns mount information, something is already mounted on `/mnt/data`. Do not mount another file system there unless that is intentional and approved.
>
> Mounting a file system over a directory hides that directory's existing contents until the file system is unmounted. The contents are not deleted, but they are inaccessible while the new file system remains mounted.

### Temporary Device-Mount Workflow

Use this workflow when you need to mount a device for the current session without making it persistent across reboots.

| Step | Task | Command |
|---|---|---|
| 1 | Identify the intended device and file system | `lsblk -f` |
| 2 | Create the mount-point directory, if necessary | `mkdir -p /mnt/data` |
| 3 | Confirm that the directory is not already a mount point | `findmnt -M /mnt/data` |
| 4 | Temporarily mount the device | `mount /dev/sdb1 /mnt/data` |
| 5 | Confirm the mount source, type, and options | `findmnt -M /mnt/data -o TARGET,SOURCE,FSTYPE,OPTIONS` |
| 6 | Confirm available space | `df -h /mnt/data` |
| 7 | Unmount the device when finished | `umount /mnt/data` |

A temporary device mount does not persist after a reboot unless you create an appropriate `/etc/fstab` entry.

### Temporary Mounts and `tmpfs`

| Type | Description | Example |
|---|---|---|
| Temporary device mount | Mount a disk, partition, USB device, or volume without adding it to `/etc/fstab`. The mount does not persist after unmounting or rebooting. | `mount /dev/sdb1 /mnt/data` |
| Persistent device mount | Configure a mount in `/etc/fstab` so the file system mounts automatically at boot. Prefer a UUID rather than a `/dev/sdX` path. | `UUID=<uuid> /mnt/data <fstype> defaults 0 0` |
| `tmpfs` | A memory-backed file system intended for temporary data. It is normally lost when the system reboots. | `mount -t tmpfs tmpfs /mnt/ramdisk` |

## STIG-Related Partitioning and Mount Options

Mount-point requirements and required options vary by applicable RHEL version, STIG release, organizational baseline, and system role. Validate the exact requirements with the current approved baseline before modifying `/etc/fstab`.

| Mount Point | Common hardening options | Notes |
|---|---|---|
| `/boot` | `nodev,nosuid` | Often configured as a separate file system. |
| `/boot/efi` | `nodev,nosuid,umask=0077,shortname=winnt` | Applicable to UEFI systems. |
| `/home` | `nodev,nosuid,noexec` | Common protection for user home directories. Confirm application requirements before applying `noexec`. |
| `/tmp` | `nodev,nosuid,noexec` | Reduces device-file, SUID/SGID, and direct-execution risk. |
| `/var` | `nodev` | Often configured as a separate file system. |
| `/var/log` | `nodev,nosuid,noexec` | Often configured separately to protect log storage. |
| `/var/log/audit` | `nodev,nosuid,noexec` | Often configured separately for audit records. |
| `/var/tmp` | `nodev,nosuid,noexec` | Commonly configured with protections similar to `/tmp`. |
| `/dev/shm` | `nodev,nosuid,noexec` | Shared-memory `tmpfs` mount. Confirm that applications do not require executable shared memory. |

| Task | Command or configuration |
|---|---|
| View current mount options | `findmnt -o TARGET,SOURCE,FSTYPE,OPTIONS` |
| View options for one mount point | `findmnt -T /tmp -o TARGET,SOURCE,FSTYPE,OPTIONS` |
| Example persistent `/tmp` entry | `UUID=1234-5678 /tmp xfs defaults,nodev,nosuid,noexec 0 0` |

## Inspecting Files and Directories

| Task | Command |
|---|---|
| Print the current working directory | `pwd` |
| Display detailed file metadata | `stat /etc/hosts` |
| Identify file type by content | `file /usr/bin/passwd` |
| Display SELinux context | `ls -Z /etc/hosts` |
| Display SELinux context for a directory itself | `ls -dZ /etc` |

## Directory Navigation

| Task | Command |
|---|---|
| Print working directory | `pwd` |
| Change to your home directory | `cd` |
| Change to a specific directory | `cd /etc` |
| Change to the parent directory | `cd ..` |
| Change to the previous directory | `cd -` |

## Directory Management

| Task | Command |
|---|---|
| Create a directory | `mkdir newdir` |
| Create parent directories as needed | `mkdir -p parent/child/grandchild` |
| Remove an empty directory | `rmdir directory` |
| Recursively remove a directory and its contents | `rm -r directory` |
| Force recursive removal without prompts | `rm -rf directory` |

{: .warning }
> **Use `rm -rf` carefully**
>
> Recursive deletion has no undo operation. Confirm the current directory, target path, and wildcard expansion before running `rm -rf`.

## Wildcards

| Wildcard | Description | Example |
|---|---|---|
| `*` | Matches any number of characters. | `*.log` |
| `?` | Matches exactly one character. | `file?.txt` |
| `[auo]` | Matches one character from the specified set. | `gr[auo]p` |
| `[a-e]` | Matches one character from the specified range. | `file[a-e]` |

## Listing Files with `ls`

| Task | Command |
|---|---|
| List directory contents | `ls` |
| Display a long listing | `ls -l` |
| Show hidden files | `ls -a` |
| Display a long listing including hidden files | `ls -la` |
| Display human-readable sizes | `ls -lh` |
| Sort by time, newest first | `ls -lt` |
| Sort by time, oldest first | `ls -lrt` |
| Sort by size, largest first | `ls -lS` |
| List contents recursively | `ls -R` |
| List the directory entry itself | `ls -d /etc/` |
| Show SELinux context | `ls -Z` |
| Display a long listing, hidden files, and SELinux context | `ls -laZ` |

## Creating, Copying, Moving, and Removing Files

| Task | Command |
|---|---|
| Create an empty file or update its timestamp | `touch newfile` |
| Copy a file | `cp source destination` |
| Copy a directory recursively | `cp -r source_dir destination_dir` |
| Copy recursively while preserving attributes | `cp -a source destination` |
| Copy with verbose output | `cp -v source destination` |
| Copy interactively with overwrite prompts | `cp -i source destination` |
| Rename or move a file | `mv oldname newname` |
| Move a file to another directory | `mv file1 /tmp/` |
| Remove a file | `rm file1` |
| Remove a file without prompting | `rm -f file1` |

{: .tip }
> **SELinux context when using `cp` and `mv`**
>
> - `cp` normally creates a new destination file with the destination directory's default SELinux context.
> - `cp -a` preserves file attributes, including SELinux context when possible.
> - `mv` normally retains the source file's SELinux context, which can cause labeling issues when moving content into locations such as `/etc` or `/var/www/html`.
> - Use `mv -Z source destination` or `cp -Z source destination` to apply the destination's default SELinux context during the operation.
> - Use `restorecon -v /path/to/file` to restore the expected context for an incorrectly labeled file.
> - Use `restorecon -Rv /path/to/directory` to recursively restore expected contexts under a directory.

## File Search with `find`

Quote wildcard patterns, such as `"*.log"`, so the shell does not expand them before `find` runs.

| Task | Command |
|---|---|
| Find files by name | `find / -name "*.conf"` |
| Find files by name, case-insensitive | `find / -iname "*.conf"` |
| Find files owned by a user | `find / -user username` |
| Find files owned by a group | `find / -group groupname` |
| Find files by UID | `find / -uid 1000` |
| Find files by GID | `find / -gid 1000` |
| Find regular files larger than 100 MB | `find / -type f -size +100M` |
| Find files smaller than 10 KB | `find / -size -10k` |
| Find files modified in the last 7 days | `find / -mtime -7` |
| Find files modified more than 30 days ago | `find / -mtime +30` |
| Find files accessed today | `find / -atime 0` |
| Find files by type | `find / -type f` or `find / -type d` |
| Find files with exact mode `0644` | `find / -perm 0644` |
| Find files with the SUID bit set | `find / -type f -perm -4000` |
| Find files with the SGID bit set | `find / -type f -perm -2000` |
| Find world-writable files | `find / -type f -perm -002` |
| Run a read-only command against matches | `find /var/log -name "*.log" -exec ls -lh {} +` |
| Request confirmation before running a command against each match | `find / -name "*.tmp" -ok rm {} \;` |
| Delete matching files | `find /tmp -type f -mtime +30 -delete` |

{: .warning }
> **Destructive `find` actions**
>
> Review search results with the same criteria before using `-delete` or `-exec rm`.
>
> For example, first run `find /tmp -type f -mtime +30 -print`, review the results, and then run the equivalent command with `-delete` if authorized.

## Hard Links and Symbolic Links

| Task | Command |
|---|---|
| Create a hard link | `ln /path/to/target /path/to/hardlink` |
| Create a symbolic link | `ln -s /path/to/target /path/to/symlink` |
| Remove a link | `rm link` |
| Inspect link target and metadata | `ls -l link` |
| Display the path stored in a symbolic link | `readlink link` |

| Feature | Hard Link: `ln` | Symbolic Link: `ln -s` |
|---|---|---|
| Points to | The same inode and underlying file data. | A file-system path. |
| Cross file systems | No. | Yes. |
| Link to directories | Normally no. | Yes. |
| If the original path is deleted | Data remains available through the hard link. | The link becomes broken or dangling. |
| Typical use | Make one file available at multiple paths without duplicating data. | Administrative links, directory links, versioned paths, and cross-file-system links. |

### Link Management Considerations

- Use absolute paths for symbolic links when practical.
- Relative symbolic-link targets are evaluated relative to the symbolic link's directory.
- Do not add a trailing slash when removing a symbolic link to a directory.
- Use `rm mylink` or `unlink mylink`; do not use `rm -rf mylink/`.
- Disk space associated with a hard-linked file is released only after the link count reaches zero and no process still has the file open.

## Archiving with `tar`

### Common Archive Operations

| Task | Command |
|---|---|
| Create an uncompressed archive | `tar -cvf archive.tar /etc` |
| Extract an uncompressed archive | `tar -xvf archive.tar` |
| Extract an archive to a directory | `tar -xvf archive.tar -C /tmp` |
| List archive contents | `tar -tvf archive.tar` |
| Append a file to an uncompressed archive | `tar -rvf archive.tar newfile` |
| Update an uncompressed archive with newer files | `tar -uvf archive.tar /home` |
| Create a gzip-compressed archive | `tar -czvf archive.tar.gz /etc` |
| Extract a gzip-compressed archive | `tar -xzvf archive.tar.gz` |
| Create a bzip2-compressed archive | `tar -cjvf archive.tar.bz2 /etc` |
| Extract a bzip2-compressed archive | `tar -xjvf archive.tar.bz2` |
| Create an xz-compressed archive | `tar -cJvf archive.tar.xz /etc` |
| Extract an xz-compressed archive | `tar -xJvf archive.tar.xz` |
| List gzip archive contents without extracting | `tar -tzvf archive.tar.gz` |

### `tar` Options

| Option | Description |
|---|---|
| `c` | Create an archive. |
| `x` | Extract an archive. |
| `t` | List archive contents. |
| `v` | Display verbose output. |
| `f` | Specify the archive filename. |
| `z` | Use gzip compression. |
| `j` | Use bzip2 compression. |
| `J` | Use xz compression. |
| `C` | Extract to a specified directory. |
| `r` | Append files to an uncompressed archive. |
| `u` | Update an uncompressed archive with newer files. |

{: .tip }
> When archiving content that requires SELinux extended attributes, use appropriate `tar` options such as `--selinux` or `--xattrs` when creating or extracting the archive.

## Compression Tools

| Use case | Recommended command | Notes |
|---|---|---|
| General-purpose single-file compression | `gzip file` | Default choice for common administrative tasks. Fast and widely supported. |
| Compress while retaining the original file | `gzip -k file` | Creates `file.gz` and retains the source file. |
| Decompress a gzip file | `gunzip file.gz` | Restores the original file. |
| Maximum compression ratio for long-term archives | `xz file` | Usually smaller than gzip, but slower and more CPU-intensive. |
| Decompress an xz file | `unxz file.xz` | Restores the original file. |
| Compatibility with existing bzip2 archives | `bzip2 file` | Primarily use when required by an established workflow or existing archive format. |
| Decompress a bzip2 file | `bunzip2 file.bz2` | Restores the original file. |

## Important Tips

- Prefer `findmnt` over `mount` when inspecting or locating mounted file systems.
- Use `findmnt -T` to identify the file system containing a path.
- Use `findmnt -M` to verify that a directory itself is an exact mount point.
- Create a mount-point directory before mounting a device when necessary.
- Prefer UUIDs rather than `/dev/sdX` names for persistent `/etc/fstab` entries.
- Use `cp -a` when you need to preserve permissions, timestamps, symbolic links, and SELinux context when possible.
- Use `rm -rf`, `find -delete`, and `find -exec rm` only after confirming the target set.
- Inspect an archive with `tar -tvf archive.tar` before extracting it.
- Do not use a trailing slash when removing a symbolic link to a directory.
- Use `gzip` as the normal default; use `xz` when smaller archive size justifies slower compression and extraction.