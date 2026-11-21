---
layout: default
title: SELinux Fundamentals
parent: Quick Reference
grand_parent: RHEL
nav_order: 17
---

# SELinux Fundamentals

## What Is SELinux?

SELinux provides **mandatory access control (MAC)** for Linux systems. Access is denied by default unless explicitly allowed by SELinux policy.

This deny-by-default model can prevent unauthorized access even when traditional Linux file permissions are incorrectly configured.

### Key Concept

SELinux applies labels, called **contexts**, to target objects such as:

- Files
- Directories
- Ports

Processes run in **domains** and must be permitted by policy to access objects with the required context type.

Example:

```text
The httpd process requires content files labeled httpd_sys_content_t.
```

## Installing `semanage`

The `semanage` command may not be installed by default.

```bash
# Identify the package that provides semanage
yum whatprovides */semanage

# Install the package
yum install policycoreutils-python-utils
```

## SELinux Modes

| Mode | Description |
|---|---|
| **Enforcing** | SELinux is fully operational and enforces policy rules. |
| **Permissive** | SELinux logs policy violations but does not block access; useful for troubleshooting. |
| **Disabled** | SELinux is not active. Changing to or from disabled mode requires a reboot. |

## SELinux Mode Management

| Task | Command or Configuration |
|---|---|
| Display current mode | `getenforce` |
| Switch to permissive mode temporarily | `setenforce 0` |
| Switch to enforcing mode temporarily | `setenforce 1` |
| Show detailed SELinux status | `sestatus` |
| Set persistent SELinux mode | Edit `/etc/sysconfig/selinux` |

Example persistent configuration:

```ini
SELINUX=enforcing
```

Other valid values:

```ini
SELINUX=permissive
SELINUX=disabled
```

{: .note }
> `sestatus -v` displays contexts for specific key files and processes defined in policy, not every object on the system. Use `ls -Z`, `ps Zaux`, or `ss -Ztul` to inspect contexts for files, processes, and ports.

## SELinux Context Labels

SELinux contexts use the following format:

```text
user:role:type
```

Example:

```text
system_u:object_r:httpd_sys_content_t:s0
```

For daily administration, focus on the **type** field, which typically ends in `_t`.

| Context Field | Purpose |
|---|---|
| `user` | SELinux user identity; typically ends in `_u`. |
| `role` | SELinux role; typically ends in `_r`. |
| `type` | Type enforcement label; typically ends in `_t`. Most relevant for routine administration. |
| `level` | MLS/MCS security level, when applicable. |

## Viewing SELinux Contexts

| Task | Command |
|---|---|
| View file or directory contexts | `ls -Z /var/www` |
| View file-context policy rules | <code>semanage fcontext -l &#124; grep httpd</code> |
| View process contexts | `ps Zaux` |
| View port contexts | `ss -Ztul` |
| View port-context policy rules | <code>semanage port -l &#124; grep ssh</code> |

### Usage Notes

Replace `httpd` or `ssh` in the `grep` command with the relevant service or context.

Examples:

```bash
semanage fcontext -l | grep ftp
semanage fcontext -l | grep httpd_sys_content_t
semanage port -l | grep ssh_port_t
```

- Search by service name, such as `httpd`, `ssh`, or `ftp`, to see related contexts.
- Search by specific context type to find an exact match.

## Setting File and Directory Contexts

{: .important }
> Use `semanage` to modify SELinux policy and `restorecon` to apply the policy to files. Avoid using `chcon` for permanent changes because its changes are lost during a relabel.

### Add and Apply a File Context Rule

```bash
# Add a recursive file-context rule to SELinux policy
semanage fcontext -a -t httpd_sys_content_t "/web(/.*)?"

# Apply the policy-defined context to the directory and contents
restorecon -Rv /web

# Verify the policy rule
semanage fcontext -l | grep /web
```

## `semanage` Parameters

| Parameter | Description |
|---|---|
| `fcontext` | Manage file and directory context rules. |
| `port` | Manage network-port labels. |
| `-a` | Add a new rule. |
| `-d` | Delete a rule. |
| `-m` | Modify an existing rule. |
| `-t TYPE` | Specify the SELinux type context. |
| `-p PROTOCOL` | Specify the network protocol, such as `tcp` or `udp`. |
| `-l` | List current rules. |

## `restorecon` Parameters

| Parameter | Description |
|---|---|
| `-R` | Apply changes recursively to directories and files. |
| `-v` | Display verbose output showing changes. |

## Common SELinux Context Types

| Context Type | Use |
|---|---|
| `httpd_sys_content_t` | Apache HTTP Server web content. |
| `httpd_sys_script_exec_t` | Apache CGI scripts. |
| `ssh_port_t` | SSH service ports. |

## Context Inheritance Rules

| Action | Result |
|---|---|
| Create a new file | Inherits the parent directory's context. |
| Copy a file with `cp` | Inherits the destination directory's context. |
| Move a file with `mv` | Retains the original source context. |

## Port Label Management

| Task | Command |
|---|---|
| List port labels related to SSH | <code>semanage port -l &#124; grep ssh</code> |
| Add SELinux label for SSH on TCP port 2022 | `semanage port -a -t ssh_port_t -p tcp 2022` |
| Delete SELinux label for SSH on TCP port 2022 | `semanage port -d -t ssh_port_t -p tcp 2022` |

## SELinux Booleans

SELinux booleans provide a method to enable or disable specific policy behavior without rewriting policy rules.

| Task | Command |
|---|---|
| List all booleans | `getsebool -a` |
| Search booleans | <code>getsebool -a &#124; grep ftp</code> |
| Set a boolean persistently | `setsebool -P ftpd_anon_write on` |

{: .note }
> SELinux booleans can change service behavior independently of the service's application configuration. Verify the intended effect before enabling a boolean persistently.

## SELinux Troubleshooting Workflow

| Step | Action |
|---:|---|
| 1 | Verify whether SELinux is causing the issue by temporarily switching to permissive mode with `setenforce 0` and testing. |
| 2 | Check the audit log for access denials. |
| 3 | Identify the source context (`scontext`) and target context (`tcontext`). |
| 4 | Apply the appropriate fix using `semanage` and `restorecon`, or set an applicable boolean. |
| 5 | Re-enable enforcing mode with `setenforce 1`. |
| 6 | Test the service to confirm the issue is resolved. |

Example:

```bash
# Temporarily test whether SELinux is the cause
setenforce 0

# Test the affected service or application

# Return SELinux to enforcing mode
setenforce 1
```

## SELinux Troubleshooting Tools

| Task | Command or Location |
|---|---|
| Primary SELinux audit log | `/var/log/audit/audit.log` |
| Search for SELinux denials | `grep AVC /var/log/audit/audit.log` |
| Install `sealert` tools | `yum install setroubleshoot-server` |
| View readable messages after `sealert` installation | `/var/log/messages` |

Examples:

```bash
# Show recent SELinux AVC denials
grep AVC /var/log/audit/audit.log | tail

# Search the audit log for a specific context type
grep httpd_sys_content_t /var/log/audit/audit.log
```

## Daily Administration Essentials

- Keep SELinux in **enforcing** mode.
- Use `semanage fcontext` and `restorecon` for persistent file and directory context changes.
- Use `semanage port` when configuring services on non-standard ports.
- Use `setsebool -P` for persistent SELinux policy behavior changes.
- Check `/var/log/audit/audit.log` for SELinux denials.
- Do not rely on `chcon` for permanent context changes because relabeling removes those changes.