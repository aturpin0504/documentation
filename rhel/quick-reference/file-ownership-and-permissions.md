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

# Recursively change user and group owner
chown -R user:group /dir
```

### File Creation Defaults

- **User owner:** The user who creates the file.
- **Group owner:** The creating user's primary group.

## Permission Resolution Order

Permission resolution stops at the first applicable match.

| Order | Check | Description |
|---|---|---|
| 1 | User owner | If you are the file owner, apply user permissions and stop. |
| 2 | User ACL | If you have a user-assigned ACL, apply the ACL and stop. |
| 3 | Group owner | If you belong to the file's group owner, apply group permissions and stop. |
| 4 | Group ACL | If you match a group ACL, apply the ACL permissions and stop. |
| 5 | Others | Apply the `others` permissions as the default fallback. |

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
| `u:user:perms` | User ACL |
| `g:group:perms` | Group ACL |
| `o::perms` | Others ACL |
| `d:` prefix | Default ACL for newly created content in a directory |

```bash
# Display ACLs
getfacl file

# Grant a user rwx access
setfacl -m u:username:rwx file

# Grant a group rx access
setfacl -m g:groupname:rx /dir

# Remove all permissions for others
setfacl -m o::- file

# Apply a group ACL recursively to existing files
setfacl -R -m g:sales:rw /data

# Set a default ACL for newly created files
setfacl -m d:g:sales:rwx /dir

# Remove a specific ACL entry
setfacl -x u:username file

# Remove all ACLs
setfacl -b file
```

{: .note }
> `ls -l` displays a `+` after the normal permission bits when ACLs are present (for example, `-rwxrwx---+`). If needed, include the `acl` mount option in `/etc/fstab` for the relevant file system.

## `umask`: Default Permissions

`umask` removes permissions from the default creation mode (`666 - umask` for files, `777 - umask` for directories).

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