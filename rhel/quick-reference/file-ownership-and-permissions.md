---
layout: default
title: Permissions & Ownership
parent: Quick Reference
grand_parent: RHEL
nav_order: 5
---

# File Ownership and Permissions

## File Ownership

```bash
# Show ownership and permissions
ls -l

# Show directory itself, not its contents
ls -ld /directory

# Change user owner
chown user file

# Recursively change user owner
chown -R user /dir

# Change user and group owner
chown user:group file

# Change only group owner
chown :group file

# Recursively change only group owner
chown -R :group /dir

# Recursively change user and group owner
chown -R user:group /dir
```

> Use the `user:group` form for user and group ownership changes. GNU `chown` also accepts a period as a separator in some cases, but the colon form is clearer and avoids ambiguity.

### File Creation Defaults

- **User owner:** Normally, the user or service account that creates the file.
- **Group owner:** Normally, the creating process's effective or primary group.
- **SGID directory exception:** If the parent directory has the SGID bit set, new files and subdirectories inherit the parent directory's group owner.
- **User-owner inheritance:** Standard Linux permissions, SGID, and POSIX ACLs cannot force newly created files to have a different user owner. To make files owned by a specific user, run the creating process as that user or change ownership after creation.

## Permission Resolution Order

The system applies the first applicable permission class. When ACLs are present, named user and group entries can affect the result.

| Order | Check | Description |
|---|---|---|
| 1 | User owner | If the process user is the file owner, apply the owner permissions. |
| 2 | Named user ACL | If an ACL entry exists for the process user, apply that entry, subject to the ACL mask. |
| 3 | Group class | If the process is a member of the owning group or any matching named-group ACLs, combine the matching group permissions and apply the ACL mask. |
| 4 | Others | If no owner, named-user ACL, or matching group-class entry applies, use the `others` permissions. |

> **ACL mask:** The ACL mask limits the effective permissions for named user ACLs, named group ACLs, and the owning group. It does not limit the file owner's permissions or the `others` permissions.

## Basic Permissions

| Permission | Symbol | Value | Files | Directories |
|---|---:|---:|---|---|
| Read | `r` | 4 | Open and read file contents | List directory entry names |
| Write | `w` | 2 | Modify file contents | Create, rename, or delete directory entries when combined with execute permission |
| Execute | `x` | 1 | Run the file as a program or script | Enter or traverse the directory; access items within it when their permissions allow |

## `chmod` Numeric Mode

### Permission Values

| Mode | Symbolic | Meaning |
|---:|---|---|
| `7` | `rwx` | Read, write, and execute (`4 + 2 + 1`) |
| `6` | `rw-` | Read and write (`4 + 2`) |
| `5` | `r-x` | Read and execute (`4 + 1`) |
| `4` | `r--` | Read only |
| `0` | `---` | No permissions |

```bash
# Set rwxr-xr-x
chmod 755 file

# Set rwxr-x--- recursively
chmod -R 750 /dir
```

## `chmod` Symbolic Mode

### Symbolic Permission Syntax

| Token | Category | Meaning |
|---|---|---|
| `u` / `g` / `o` / `a` | Target | User owner, group owner, others, or all |
| `+` / `-` / `=` | Operator | Add permission, remove permission, or set exact permissions |
| `r` / `w` / `x` / `X` | Permission | Read, write, execute, or conditional execute |

```bash
# Add execute permission for user
chmod u+x file

# Add write permission for group
chmod g+w file

# Remove read permission from others
chmod o-r file

# Add read permission for all
chmod a+r file

# Add execute permission for all
chmod +x file

# Set exact permissions
chmod u=rwx,g=rx,o= file

# Recursively add read for others and execute only on directories
# or files that are already executable
chmod -R o+rX /data
```

Uppercase `X` sets execute permission only for directories and files that already have execute permission for at least one permission class.

## Advanced Permissions

| Permission | Numeric | Symbolic | Effect on Files | Effect on Directories |
|---|---:|---|---|---|
| SUID | 4 | `u+s` | Program executes with the file owner's effective permissions | Generally no useful effect |
| SGID | 2 | `g+s` | Program executes with the file group's effective permissions | New files and subdirectories inherit the directory's group owner |
| Sticky bit | 1 | `+t` | No effect | Users can delete or rename only their own files, unless they own the directory or have elevated privileges |

```bash
# Set SUID on an executable file
chmod 4755 file
chmod u+s file

# Set SGID on a shared directory
chmod 2770 /dir
chmod g+s /dir

# Set the sticky bit on a public shared directory
chmod 1777 /dir
chmod +t /dir
```

When viewing permissions with `ls -l`:

- SUID appears as `s` or `S` in the user execute position.
- SGID appears as `s` or `S` in the group execute position.
- The sticky bit appears as `t` or `T` in the others execute position.

Uppercase `S` or `T` indicates that the special bit is set but the corresponding execute permission is not set.

## Access Control Lists (ACLs)

ACLs provide more granular permissions than the standard user/group/other permission model. They allow permissions for additional named users and groups without changing the file owner or owning group.

### ACL Syntax

| Syntax | Meaning |
|---|---|
| `u:user:perms` | Named user ACL entry |
| `g:group:perms` | Named group ACL entry |
| `u::perms` | File owner's ACL permissions |
| `g::perms` | Owning group's ACL permissions |
| `o::perms` | Others ACL permissions |
| `m::perms` | ACL mask; limits effective permissions for the owning group and named user/group ACL entries |
| `d:` prefix | Default ACL entry inherited by newly created content in a directory |

> **Access ACLs** control access to an existing file or directory. **Default ACLs** can be applied only to directories and are inherited as access ACLs by newly created files and subdirectories.

```bash
# Display ACLs, including effective permissions
getfacl file

# Grant a user read, write, and execute access
setfacl -m u:username:rwx file

# Grant a group read and execute access to a directory
setfacl -m g:groupname:rx /dir

# Set the ACL mask explicitly
setfacl -m m::rw- file

# Remove all permissions for others
setfacl -m o::--- file

# Apply a group ACL recursively:
# rw for files; add x only to directories and already-executable files
setfacl -R -m g:sales:rwX /data

# Set a named-group default ACL for newly created content
setfacl -m d:g:sales:rwx /dir

# Set default owner, group, and others permissions for new content
setfacl -d -m u::rwx,g::rwx,o::--- /dir

# Remove a specific ACL entry
setfacl -x u:username file

# Remove all default ACL entries from a directory
setfacl -k /dir

# Remove extended ACL entries
setfacl -b file
```

{: .note }
> `ls -l` displays a `+` after the normal permission bits when an extended ACL is present, for example `-rwxrwx---+`. Use `getfacl file` or `getfacl /directory` to view ACL entries and effective permissions.

### Default ACLs, SGID, and `umask`

SGID, default ACLs, and `umask` control different parts of file creation.

| Feature | Controls | Does Not Control |
|---|---|---|
| SGID on a directory | Group ownership of newly created files and subdirectories | User ownership or inherited permissions |
| Default ACL on a directory | Inherited access permissions for newly created files and subdirectories | User ownership or group ownership |
| `umask` | Permissions removed from new files and directories when no default ACL applies | Ownership or permissions on existing content |

When a directory does not have a default ACL, the creating process's `umask` modifies the requested creation permissions.

When a directory has a default ACL, new content inherits that ACL instead of applying `umask` in the normal manner. The creating program's requested mode can still restrict the final permissions.

For example, ordinary files are normally created with a requested mode of `666`, so they do not automatically receive execute permission even if the directory's default ACL includes `x`. New directories are normally created with a requested mode of `777` and can inherit execute permission.

> **Recommendation:** Use SGID alone for a simple shared directory when all creators use a collaborative `umask`. Add a default ACL when you need consistent inherited permissions across users or services with different `umask` values.

## `umask`: Default Permissions

`umask` removes permission bits from the default creation mode. For regular files, the starting mode is commonly `666`; for directories, it is commonly `777`.

Conceptually:

- Files: requested mode `666`, then permissions are restricted by `umask`.
- Directories: requested mode `777`, then permissions are restricted by `umask`.

> `umask` is a bit mask, not ordinary arithmetic subtraction. For example, a `umask` of `027` commonly results in file permissions of `640` and directory permissions of `750`. If the parent directory has a default ACL, that ACL is used instead of applying `umask` in the normal manner.

| `umask` | Default File Permissions | Default Directory Permissions |
|---:|---:|---:|
| `022` | `644` | `755` |
| `027` | `640` | `750` |
| `077` | `600` | `700` |
| `002` | `664` | `775` |

```bash
# Display current umask
umask

# Display umask in symbolic form
umask -S

# Set umask for the current shell session
umask 022
umask 027
umask 077
```

### Configure `umask` Persistently

| Scope | Typical Bash Configuration Location |
|---|---|
| System-wide Bash shell setting | `/etc/bashrc` |
| System-wide login-shell setting | A file such as `/etc/profile.d/umask.sh` |
| One user's interactive Bash shells | `~/.bashrc` |
| One user's Bash login shells | `~/.bash_profile` |

For system-wide configuration, follow the organization's approved RHEL baseline and account-management policy. A typical baseline uses a more restrictive `umask` for privileged accounts and may use `022` or `027` for standard accounts, depending on collaboration and security requirements.

## Extended Attributes

| Attribute | Symbol | Effect |
|---|---|---|
| Immutable | `i` | File cannot be modified, deleted, renamed, or linked. |
| Append-only | `a` | Existing file data cannot be changed; data can only be appended. |
| No atime update | `A` | Access-time updates are disabled. |

```bash
# List file attributes
lsattr file

# Set or remove immutable attribute
chattr +i file
chattr -i file

# Set or remove append-only attribute
chattr +a file
chattr -a file

# Disable access-time updates
chattr +A file
```

> Extended attributes and filesystem attributes depend on the filesystem type and mount configuration. Verify support before relying on them in an operational workflow.

## Common Permission Scenarios

### Create a Shared Group Directory

Use SGID when new content should inherit the directory's group owner.

```bash
# Create the shared directory
mkdir /data/sales

# Set the shared group owner
chown root:sales /data/sales

# Grant owner and group full access; enable SGID group inheritance
chmod 2770 /data/sales
```

New files and subdirectories created in `/data/sales` inherit the `sales` group.

SGID alone is sufficient when users and services create files with a `umask` that preserves the required group permissions. To enforce inherited group access despite differing `umask` values, add a default ACL:

```bash
# Set default permissions for newly created files and directories
setfacl -d -m u::rwx,g::rwx,o::--- /data/sales
```

Verify ownership, SGID, and ACL settings:

```bash
ls -ld /data/sales
getfacl /data/sales
```

### Create a Public Shared Directory

```bash
# Create the directory
mkdir /data/public

# Allow all users access while preventing deletion of others' files
chmod 1777 /data/public
```

The sticky bit allows users to create files in the directory but prevents them from deleting or renaming files owned by other users.

### Grant Cross-Group Access with ACLs

```bash
# Allow the account group to read and enter the sales directory
setfacl -m g:account:rx /data/sales

# Allow the sales group to read and enter the account directory
setfacl -m g:sales:rx /data/account
```

### Grant a User Access Without Changing Ownership

```bash
# Allow a specific user to read and modify a file
setfacl -m u:username:rw- file

# Allow a specific user to enter and modify content in a directory
setfacl -m u:username:rwx /data/project
```

Verify the result:

```bash
getfacl file
getfacl /data/project
```