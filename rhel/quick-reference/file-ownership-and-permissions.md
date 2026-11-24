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

# Change user and group owner (colon or period works as separator)
chown user:group file
chown user.group file

# Change only group owner
chown :group file

# Recursively change only group owner
chown -R :group /dir

# Recursively change user and group owner
chown -R user:group /dir
```

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
| Read | `r` | 4 | Open and read file contents | List directory contents |
| Write | `w` | 2 | Modify file contents | Create or delete directory entries |
| Execute | `x` | 1 | Run the file as a program or script | Enter the directory with `cd` |

## `chmod` Numeric Mode

### Permission Values

| Mode | Symbolic | Meaning |
|---:|---|---|
| `7` | `rwx` | Read, write, and execute (`4 + 2 + 1`) |
| `6` | `rw-` | Read and write (`4 + 2`) |
| `5` | `r-x` | Read and execute (`4 + 1`) |
| `4` | `r--` | Read only |

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

# Recursively add read for others and execute only on directories or already-executable files
chmod -R o+rX /data
```

Uppercase `X` sets execute permission only for directories and files that already have execute permission for at least one permission class.

## Advanced Permissions

| Permission | Numeric | Symbolic | Effect on Files | Effect on Directories |
|---|---:|---|---|---|
| SUID | 4 | `u+s` | Executes with the file owner's permissions | No effect |
| SGID | 2 | `g+s` | Executes with the file group's permissions | New files inherit the directory's group owner |
| Sticky bit | 1 | `+t` | No effect | Users cannot delete files owned by other users |

```bash
# Set SUID with rwxr-xr-x (numeric and symbolic)
chmod 4755 file
chmod u+s file

# Set SGID with rwxrwx--- (numeric and symbolic)
chmod 2770 /dir
chmod g+s /dir

# Set sticky bit with rwxrwxrwx (numeric and symbolic)
chmod 1777 /dir
chmod +t /dir
```

When viewing permissions with `ls -l`, a sticky bit appears as `t` in the execute position for `others`.

## Access Control Lists (ACLs)

ACLs provide more granular permissions than the standard user/group/other permission model.

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

# Set the ACL mask explicitly
setfacl -m m::rw- file

# Set a default ACL for newly created files
setfacl -m d:g:sales:rwx /dir

# Set default owner, group, and others permissions for new content
setfacl -d -m u::rwx,g::rwx,o::--- /dir

# Remove all default ACL entries from a directory
setfacl -k /dir

# Grant a user rwx access
setfacl -m u:username:rwx file

# Grant a group rx access
setfacl -m g:groupname:rx /dir

# Remove all permissions for others
setfacl -m o::- file

# Apply a group ACL recursively to existing files
setfacl -R -m g:sales:rw /data

# Remove a specific ACL entry
setfacl -x u:username file

# Remove all ACLs
setfacl -b file
```

{: .note }
> `ls -l` displays a `+` after the normal permission bits when ACLs are present (for example, `-rwxrwx---+`). If needed, include the `acl` mount option in `/etc/fstab` for the relevant file system.

### Default ACLs, SGID, and `umask`

SGID, default ACLs, and `umask` control different parts of file creation:

| Feature | Controls | Does Not Control |
|---|---|---|
| SGID on a directory | Group ownership of newly created files and subdirectories | User ownership or inherited permissions |
| Default ACL on a directory | Inherited access permissions for newly created files and subdirectories | User ownership or group ownership |
| `umask` | Permissions removed from new files and directories when no default ACL applies | Ownership or permissions on existing content |

When a directory does not have a default ACL, the creating process's `umask` modifies the requested creation permissions.

When a directory has a default ACL, new content inherits that ACL instead of applying `umask` in the normal manner. The creating program's requested mode can still restrict the final permissions. For example, ordinary files are normally created with a requested mode of `666`, so they do not automatically receive execute permission even if the directory's default ACL includes `x`.

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

# Set umask for the current shell session
umask 022
umask 027
umask 077
```

### Configure `umask` Persistently

| Scope | Location |
|---|---|
| Default system setting | `/etc/bashrc` |
| All users | Create a file such as `/etc/profile.d/umask.sh` |
| One user | Add the setting to the user's `~/.bash_profile` |

Common practice is `umask 027` for `root` and `umask 022` (or `027` in hardened baselines) for normal users.

## Extended Attributes

| Attribute | Symbol | Effect |
|---|---|---|
| Immutable | `i` | File cannot be modified, deleted, or renamed. |
| Append-only | `a` | Data can only be appended to the file. |
| No atime update | `A` | Access-time updates are disabled, which can improve performance. |

```bash
# List file attributes
lsattr file

# Set or remove immutable attribute
chattr +i file
chattr -i file

# Set append-only attribute
chattr +a file

# Disable access-time updates
chattr +A file
```

If needed, add the `user_xattr` mount option in `/etc/fstab` for the applicable file system.

## Common Permission Scenarios

### Create a Shared Group Directory

```bash
# Create the shared directory
mkdir /data/sales

# Set group ownership
chown root:sales /data/sales

# Enable SGID (so new files inherit the sales group) and grant user/group access
chmod 2770 /data/sales
```

### Create a Public Shared Directory

```bash
# Create the directory
mkdir /data/public

# Allow all users access while preventing deletion of others' files
chmod 1777 /data/public
```

### Grant Cross-Group Access with ACLs

```bash
# Allow the account group to read and enter the sales directory
setfacl -m g:account:rx /data/sales

# Allow the sales group to read and enter the account directory
setfacl -m g:sales:rx /data/account
```