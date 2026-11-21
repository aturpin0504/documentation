---
layout: default
title: File Ownership and Permissions
parent: Quick Reference
grand_parent: RHEL
nav_order: 5
---

# File Ownership and Permissions

## File Ownership

| Task | Command |
|---|---|
| Show ownership and permissions | `ls -l` |
| Show directory itself, not its contents | `ls -ld /directory` |
| Change user owner | `chown user file` |
| Recursively change user owner | `chown -R user /dir` |
| Change user and group owner | `chown user:group file` |
| Change only group owner | `chown :group file` |
| Recursively change user and group owner | `chown -R user:group /dir` |

### File Creation Defaults

- **User owner:** The user who creates the file.
- **Group owner:** The creating user's primary group.
- A period (`.`) or colon (`:`) can separate user and group names in `chown` syntax.

```bash
chown user:group file
chown user.group file
```

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

| Permissions | Command |
|---|---|
| `rwxr-xr-x` | `chmod 755 file` |
| `rwxr-x---`, recursively | `chmod -R 750 /dir` |

### Permission Calculation

```text
r = 4
w = 2
x = 1

rwx = 7
rw- = 6
r-x = 5
r-- = 4
```

## `chmod` Symbolic Mode

| Task | Command |
|---|---|
| Add execute permission for user | `chmod u+x file` |
| Add write permission for group | `chmod g+w file` |
| Remove read permission from others | `chmod o-r file` |
| Add read permission for all | `chmod a+r file` |
| Add execute permission for all | `chmod +x file` |
| Set exact permissions | `chmod u=rwx,g=rx,o= file` |
| Recursively add read permission for others and execute only where appropriate | `chmod -R o+rX /data` |

### Symbolic Permission Syntax

```text
u = user owner
g = group owner
o = others
a = all

+ = add permission
- = remove permission
= = set exact permissions

r = read
w = write
x = execute
```

### Uppercase `X`

Uppercase `X` sets execute permission only for:

- Directories.
- Files that already have execute permission for at least one permission class.

```bash
chmod -R o+rX /data
```

## Advanced Permissions

| Permission | Numeric | Symbolic | Effect on Files | Effect on Directories |
|---|---:|---|---|---|
| SUID | 4 | `u+s` | Executes with the file owner's permissions | No effect |
| SGID | 2 | `g+s` | Executes with the file group's permissions | New files inherit the directory's group owner |
| Sticky bit | 1 | `+t` | No effect | Users cannot delete files owned by other users |

| Task | Command |
|---|---|
| Set SUID with `rwxr-xr-x` | `chmod 4755 file` |
| Set SGID with `rwxrwx---` | `chmod 2770 /dir` |
| Set sticky bit with `rwxrwxrwx` | `chmod 1777 /dir` |
| Add SUID symbolically | `chmod u+s file` |
| Add SGID symbolically | `chmod g+s /dir` |
| Add sticky bit symbolically | `chmod +t /dir` |

## Access Control Lists (ACLs)

ACLs provide more granular permissions than the standard user/group/other permission model.

| Task | Command |
|---|---|
| Display ACLs | `getfacl file` |
| Grant a user `rwx` access | `setfacl -m u:username:rwx file` |
| Grant a group `rx` access | `setfacl -m g:groupname:rx /dir` |
| Remove all permissions for others | `setfacl -m o::- file` |
| Apply a group ACL recursively to existing files | `setfacl -R -m g:sales:rw /data` |
| Set a default ACL for newly created files | `setfacl -m d:g:sales:rwx /dir` |
| Remove an ACL entry | `setfacl -x u:username file` |
| Remove all ACLs | `setfacl -b file` |

### ACL Syntax

| Syntax | Meaning |
|---|---|
| `u:user:perms` | User ACL |
| `g:group:perms` | Group ACL |
| `o::perms` | Others ACL |
| `d:` prefix | Default ACL for newly created content in a directory |

{: .note }
> `ls -l` displays a `+` after the normal permission bits when ACLs are present.

Example:

```text
-rwxrwx---+ 1 root sales 0 Nov 19 12:00 shared-file
```

If needed, include the `acl` mount option in `/etc/fstab` for the relevant file system.

## `umask`: Default Permissions

`umask` removes permissions from the default creation mode:

```text
Files:       666 - umask
Directories: 777 - umask
```

| `umask` | Default File Permissions | Default Directory Permissions |
|---:|---:|---:|
| `022` | `644` | `755` |
| `027` | `640` | `750` |
| `077` | `600` | `700` |
| `002` | `664` | `775` |

| Task | Command |
|---|---|
| Display current umask | `umask` |
| Set files to `644` and directories to `755` | `umask 022` |
| Set files to `640` and directories to `750` | `umask 027` |
| Set files to `600` and directories to `700` | `umask 077` |

### Configure `umask`

| Scope | Location |
|---|---|
| Default system setting | `/etc/bashrc` |
| All users | Create a file such as `/etc/profile.d/umask.sh` |
| One user | Add the setting to the user's `~/.bash_profile` |

Common practice:

```text
root:         umask 027
normal users: umask 022
```

## Extended Attributes

| Task | Command |
|---|---|
| List file attributes | `lsattr file` |
| Set immutable attribute | `chattr +i file` |
| Remove immutable attribute | `chattr -i file` |
| Set append-only attribute | `chattr +a file` |
| Disable access-time updates | `chattr +A file` |

| Attribute | Symbol | Effect |
|---|---|---|
| Immutable | `i` | File cannot be modified, deleted, or renamed. |
| Append-only | `a` | Data can only be appended to the file. |
| No atime update | `A` | Access-time updates are disabled, which can improve performance. |

If needed, add the `user_xattr` mount option in `/etc/fstab` for the applicable file system.

## Common Permission Scenarios

### Create a Shared Directory

```bash
# Create the shared directory
mkdir /data/sales

# Set group ownership
chown root:sales /data/sales

# Enable SGID and grant full user/group access
chmod 2770 /data/sales
```

The SGID bit ensures new files and directories inherit the `sales` group.

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

## Important Tips

- Use SGID on shared directories with `chmod g+s` or a numeric mode such as `2770`.
- Use ACLs for complex or cross-group permission requirements.
- Use `umask 027` when more restrictive default permissions are required.
- The sticky bit prevents users from deleting or renaming files owned by other users in a shared directory.
- When viewing permissions with `ls -l`, a sticky bit appears as `t` in the execute position for `others`.