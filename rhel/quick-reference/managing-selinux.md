---
layout: default
title: SELinux
parent: Quick Reference
grand_parent: RHEL
nav_order: 17
---

# SELinux Fundamentals

## What Is SELinux?

SELinux provides **mandatory access control (MAC)** for Linux systems. Access is denied by default unless explicitly allowed by SELinux policy—even when traditional Linux file permissions are permissive.

SELinux applies labels (**contexts**) to files, directories, and ports. Processes run in **domains** and must be permitted by policy to access objects with the required context type (for example, `httpd` requires content files labeled `httpd_sys_content_t`).

## Installing `semanage`

The `semanage` command may not be installed by default:

```bash
# Identify and install the package that provides semanage
yum whatprovides */semanage
yum install policycoreutils-python-utils
```

## SELinux Modes

| Mode | Description |
|---|---|
| **Enforcing** | SELinux is fully operational and enforces policy rules. |
| **Permissive** | SELinux logs policy violations but does not block access; useful for troubleshooting. |
| **Disabled** | SELinux is not active. Changing to or from disabled mode requires a reboot. |

```bash
# Display current mode and detailed SELinux status
getenforce
sestatus

# Switch to permissive (0) or enforcing (1) mode temporarily
setenforce 0
setenforce 1
```

To set the persistent mode, edit `/etc/sysconfig/selinux`:

```ini
SELINUX=enforcing   # Or permissive / disabled
```

{: .note }
> `sestatus -v` displays contexts for specific key files and processes defined in policy, not every object on the system. Use `ls -Z`, `ps Zaux`, or `ss -Ztul` to inspect contexts for files, processes, and ports.

## SELinux Context Labels

SELinux contexts use the format `user:role:type[:level]` (for example, `system_u:object_r:httpd_sys_content_t:s0`). For daily administration, focus on the **type** field, which typically ends in `_t`.

| Context Field | Purpose |
|---|---|
| `user` | SELinux user identity; typically ends in `_u`. |
| `role` | SELinux role; typically ends in `_r`. |
| `type` | Type enforcement label; typically ends in `_t`. Most relevant for routine administration. |
| `level` | MLS/MCS security level, when applicable. |

### Viewing SELinux Contexts

```bash
# View file/directory, process, and port contexts
ls -Z /var/www
ps Zaux
ss -Ztul

# Search file-context and port-context policy rules
semanage fcontext -l | grep httpd
semanage fcontext -l | grep httpd_sys_content_t
semanage port -l | grep ssh
semanage port -l | grep ssh_port_t
```

## Setting File and Directory Contexts

{: .important }
> Use `semanage` to modify SELinux policy and `restorecon` to apply the policy to files. Avoid using `chcon` for permanent changes because its changes are lost during a relabel.

```bash
# Add a recursive file-context rule to SELinux policy
semanage fcontext -a -t httpd_sys_content_t "/web(/.*)?"

# Apply the policy-defined context to the directory and contents
restorecon -Rv /web

# Verify the policy rule
semanage fcontext -l | grep /web
```

### `semanage` and `restorecon` Parameters

| Command / Parameter | Description |
|---|---|
| `semanage fcontext` | Manage file and directory context rules. |
| `semanage port` | Manage network-port labels. |
| `-a` / `-d` / `-m` | Add, delete, or modify a rule. |
| `-t TYPE` | Specify the SELinux type context. |
| `-p PROTOCOL` | Specify the network protocol (`tcp` or `udp`). |
| `-l` | List current rules. |
| `restorecon -R -v` | Apply policy contexts recursively (`-R`) with verbose output (`-v`). |

### Common SELinux Context Types

| Context Type | Use |
|---|---|
| `httpd_sys_content_t` | Apache HTTP Server web content. |
| `httpd_sys_script_exec_t` | Apache CGI scripts. |
| `ssh_port_t` | SSH service ports. |

### Context Inheritance Rules

| Action | Result |
|---|---|
| Create a new file | Inherits the parent directory's context. |
| Copy a file with `cp` | Inherits the destination directory's context. |
| Move a file with `mv` | Retains the original source context. |

## Port Label Management

```bash
# List port labels related to SSH
semanage port -l | grep ssh

# Add or delete an SELinux label for SSH on TCP port 2022
semanage port -a -t ssh_port_t -p tcp 2022
semanage port -d -t ssh_port_t -p tcp 2022
```

## SELinux Booleans

SELinux booleans enable or disable specific policy behavior without rewriting policy rules.

```bash
# List all booleans or search by service
getsebool -a
getsebool -a | grep ftp

# Set a boolean persistently
setsebool -P ftpd_anon_write on
```

{: .note }
> SELinux booleans can change service behavior independently of the service's application configuration. Verify the intended effect before enabling a boolean persistently.

## SELinux Troubleshooting

| Step | Action |
|---:|---|
| 1 | Verify whether SELinux is causing the issue by temporarily switching to permissive mode with `setenforce 0` and testing. |
| 2 | Check `/var/log/audit/audit.log` for `AVC` access denials. |
| 3 | Identify the source context (`scontext`) and target context (`tcontext`). |
| 4 | Apply the appropriate fix using `semanage` and `restorecon`, or set an applicable boolean. |
| 5 | Re-enable enforcing mode with `setenforce 1`. |
| 6 | Test the service to confirm the issue is resolved. |

```bash
# Temporarily test whether SELinux is the cause, then return to enforcing mode
setenforce 0
# ... test the affected service ...
setenforce 1

# Search /var/log/audit/audit.log for SELinux denials or specific context types
grep AVC /var/log/audit/audit.log
grep AVC /var/log/audit/audit.log | tail
grep httpd_sys_content_t /var/log/audit/audit.log

# Install sealert tools (outputs readable analysis to /var/log/messages)
yum install setroubleshoot-server
```