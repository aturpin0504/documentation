---
layout: default
title: fapolicyd
parent: Quick Reference
grand_parent: RHEL
nav_order: 19
---

# fapolicyd: File Access Policy Daemon for RHEL 8.6–10

## Overview

`fapolicyd` is an application allow-listing framework that controls application execution and file access based on trust and administrator-defined policy.

At runtime, `fapolicyd` consumes kernel `fanotify` events: when a process attempts to open or execute a file, the kernel pauses the operation and queries `fapolicyd`, which evaluates the request against its trust database (`RPMDB` and `File` backends) and compiled ruleset before returning an allow or deny decision.

Administrators can define allow, deny, syslog, and audit rules based on attributes such as:

- File path or directory
- SHA-256 hash
- MIME type
- Executable path
- Execution pattern (such as dynamic linker invocation)
- User or group identity
- Trust status

Applications installed through DNF and registered in the RPM database are normally trusted. Installing `fapolicyd` also installs `rpm-plugin-fapolicyd` (which monitors DNF/RPM transactions and notifies `fapolicyd` when RPM database content changes) and `fapolicyd-selinux`.

{: .note }
> This guide uses the `rules.d/`, `trust.d/`, `fagenrules`, and modern `fapolicyd-cli` workflow available in RHEL 8.6 and later. For earlier RHEL 8 releases, review the installed `fapolicyd` documentation and configuration layout before applying these procedures.

{: .note }
> When RHEL is deployed using a DISA security profile, verify whether `fapolicyd` has already been installed and configured by the applicable baseline or automation before making changes.

## Configuration Files

| File or Directory | Purpose |
|---|---|
| `/etc/fapolicyd/fapolicyd.conf` | Main daemon configuration file. Contains options for permissive mode, integrity checking, `syslog_format`, performance, and watched filesystems. |
| `/etc/fapolicyd/fapolicyd.trust` | Default list of trusted files outside RPM management. |
| `/etc/fapolicyd/trust.d/` | Additional named trusted-file lists. |
| `/etc/fapolicyd/rules.d/` | Component files containing allow, deny, and audit rules. |
| `/etc/fapolicyd/compiled.rules` | Compiled rule set generated from `rules.d/` by `fagenrules`. |
| `/var/lib/fapolicyd/` | fapolicyd trust database location. Do not remove this directory. |

## Rule Order

{: .important }
> Rule files in `/etc/fapolicyd/rules.d/` are processed in numerical filename order. Evaluation uses a **first-match-wins** model, so an allow rule must appear before the deny rule it is intended to override.

Typical default rule-file ordering includes:

| Prefix | Typical Purpose |
|---|---|
| `10` | Language-related rules |
| `20` | Dracut-related rules |
| `21` | Updater rules |
| `30` | Pattern rules |
| `40` | ELF rules |
| `41` | Shared-object rules |
| `42` | Trusted ELF rules |
| `70` | Trusted language rules |
| `72` | Shell rules |
| `90` | Deny-execution rules |
| `95` | Allow-open rules |

Use a custom rule file such as:

```text
/etc/fapolicyd/rules.d/80-local.rules
```

This places local allow rules before the typical `90-deny-execute.rules` file.

```bash
# Review source rule files in processing order
ls -1 /etc/fapolicyd/rules.d/

# Review active compiled rules and their order
fapolicyd-cli --list
```

## Ownership, Permissions, and SELinux Context

fapolicyd rules, trust files, and configuration files are security-sensitive. They must not be writable by unprivileged users or application accounts.

When editing existing files in `/etc/fapolicyd/` as `root`, ownership, permissions, and SELinux context normally remain unchanged. Verify metadata when files are copied, restored, created by automation, or moved from another directory.

```bash
# Review ownership, mode, and SELinux context for fapolicyd configuration
ls -lZ /etc/fapolicyd/
ls -lZ /etc/fapolicyd/rules.d/
ls -lZ /etc/fapolicyd/trust.d/

# Show expected SELinux context for a new rule file
matchpathcon /etc/fapolicyd/rules.d/80-local.rules

# Show expected SELinux context for a new trust file
matchpathcon /etc/fapolicyd/trust.d/myapp

# Restore policy-defined SELinux labels after copying or restoring files
restorecon -RFv /etc/fapolicyd/
```

### New Rule and Trust Files

When creating a new file, copy the ownership and permissions of a comparable existing file instead of guessing.

```bash
# Inspect a similar existing rule file
ls -lZ /etc/fapolicyd/rules.d/90-deny-execute.rules

# Inspect a similar existing trust file
ls -lZ /etc/fapolicyd/trust.d/

# Create a local rule file
touch /etc/fapolicyd/rules.d/80-local.rules

# Match the owner and group used by existing local rule files
chown root:fapolicyd /etc/fapolicyd/rules.d/80-local.rules

# Restore the expected SELinux context
restorecon -v /etc/fapolicyd/rules.d/80-local.rules
```

{: .warning }
> Do not make fapolicyd configuration, trust, or rule files writable by non-root users, application accounts, or shared groups. An unauthorized change to a trust file or allow rule can weaken application allow-listing controls.

### Application File Paths

Trusting or allowing an application path in fapolicyd does not change the path's Linux ownership, permissions, or SELinux context.

For example, a web application under `/var/www/html/` must still have ownership, permissions, and SELinux labels that allow the web server to read or modify the necessary content.

```bash
# Review metadata for an application directory
ls -ldZ /var/www/html/password/

# Review files below the application directory
ls -lZ /var/www/html/password/

# Restore standard SELinux labels for content under /var/www/html/
restorecon -RFv /var/www/html/password/
```

{: .note }
> Use `restorecon` for files under standard paths such as `/var/www/html/`. If an application uses a nonstandard path, define an approved persistent SELinux file-context rule with `semanage fcontext`, then apply it with `restorecon`.

### Generated Files

Do not manually edit the generated rule file:

```text
/etc/fapolicyd/compiled.rules
```

Modify source files under `/etc/fapolicyd/rules.d/`, then regenerate and reload the compiled rules:

```bash
fagenrules --check
fagenrules --load
fapolicyd-cli --reload-rules
```

## Trust Sources and Rule Types

| Method | Best Use Case |
|---|---|
| RPM database trust | Packages installed through DNF or RPM. |
| Trust file entry | Static custom file or application tree outside RPM management. |
| Custom path rule | Execution needs to be allowed only from a specific path. |
| Directory rule | Dynamic-content directory where file checksums change frequently. |
| SHA-256 hash rule | A specific file must match an approved checksum. |
| User, group, or executable condition | Access must be limited to a specific account or parent executable. |

{: .note }
> For static custom applications, trust entries are generally preferred over custom rules because they are simpler and more efficient. Use custom rules when access requires additional conditions such as executable path, user, group, directory, MIME type, or SHA-256 hash.

## Enabling `fapolicyd`

Install the package if it is not already installed:

```bash
dnf install -y fapolicyd
```

For initial testing, use permissive mode first.

```ini
# /etc/fapolicyd/fapolicyd.conf
permissive = 1
```

Then enable and start the service:

```bash
systemctl enable --now fapolicyd
systemctl status fapolicyd
```

In permissive mode, `fapolicyd` records actions that would otherwise be denied without blocking them.

```bash
# Search recent fapolicyd audit events
ausearch -ts recent -m fanotify -i

# Review service messages from the current boot
journalctl -u fapolicyd -b
```

After validating required applications, return to enforcing mode:

```ini
# /etc/fapolicyd/fapolicyd.conf
permissive = 0
```

```bash
systemctl restart fapolicyd
systemctl status fapolicyd
```

## Marking Files as Trusted

Use trust files for known, static executables or application directories outside RPM management.

### Add Trusted Files

```bash
# Add a single file to the default trust file
fapolicyd-cli --file add /opt/myapp/bin/tool

# Add a single file to a named trust file
fapolicyd-cli --file add /opt/myapp/bin/tool \
  --trust-file myapp

# Add existing files under a directory to a named trust file
fapolicyd-cli --file add /opt/myapp/ \
  --trust-file myapp

# Update the active trust database
fapolicyd-cli --update
```

The named trust-file example creates or updates:

```text
/etc/fapolicyd/trust.d/myapp
```

### Review Trusted Files

```bash
# Review the default trust file
cat /etc/fapolicyd/fapolicyd.trust

# Review a named trust file
cat /etc/fapolicyd/trust.d/myapp

# Review all named trust files
find /etc/fapolicyd/trust.d/ -maxdepth 1 -type f \
  -print -exec cat {} \;

# Search the active trust database for a path
fapolicyd-cli --dump-db | grep -F /opt/myapp

# Validate trust-database entries
fapolicyd-cli --check-trustdb
```

### Update Trust After a File Changes

When the content of a trusted file changes, its checksum changes and it is no longer trusted until the trust entry is refreshed.

```bash
# Refresh a changed file or directory
fapolicyd-cli --file update /opt/myapp/bin/tool

# Refresh all file-trust entries
fapolicyd-cli --file update

# Update the active trust database
fapolicyd-cli --update
```

### Remove a Trust Entry

Remove the applicable entry with `fapolicyd-cli` or by editing the trust file, then update the database (or restart the service).

```bash
# Remove a file from a named trust file
fapolicyd-cli --file delete /opt/myapp/bin/tool --trust-file myapp
fapolicyd-cli --update

# Or edit a named trust file / default trust file directly
vi /etc/fapolicyd/trust.d/myapp
vi /etc/fapolicyd/fapolicyd.trust

# Restart after manual edits
systemctl restart fapolicyd
```

{: .warning }
> Trusting a directory adds trust for the files present when the trust entry is created. Files added or changed later require a trust-database refresh. Do not use trust entries for directories containing untrusted, user-writable, or frequently changing content.

## Adding Custom Rules

Use custom rules when a trust entry is too broad or does not provide the required conditions.

### Rule Structure

Rules use Subject/Object notation (similar to SELinux) with a **first-match-wins** evaluation order:

```text
decision permission subject : object
```

Example:

```text
allow perm=execute exe=/usr/bin/bash trust=1 : path=/opt/myapp/bin/task.sh ftype=text/x-shellscript trust=0
```

| Element | Meaning |
|---|---|
| `allow` / `allow_syslog` / `allow_audit` | Permit the action (optionally logging to syslog or audit). |
| `deny` / `deny_syslog` / `deny_audit` | Deny the action (optionally logging to syslog or audit). |
| `perm=execute` / `perm=open` / `perm=any` | Match execution attempts, file opens, or any access. |
| `all` | Wildcard matching any subject or object. |
| `exe=/usr/bin/bash` | The parent executable attempting the action. |
| `trust=1` / `trust=0` | Match trusted (`1`) or untrusted (`0`) files; valid on both subject and object sides. |
| `path=` | Matches a specific file path. |
| `dir=` | Matches a directory path. A trailing `/` applies recursively. |
| `ftype=` | Matches a file MIME type (check with `fapolicyd-cli --ftype <path>`). |
| `pattern=` | Matches an execution pattern (`normal`, `static`, `ld_preload`, `ld_so`). |
| `sha256hash=` | Matches a file with the specified SHA-256 hash. |

Before writing `ftype=` rules, verify the exact MIME type `fapolicyd` assigns to a file (which can differ from `file`):

```bash
fapolicyd-cli --ftype /opt/myapp/bin/task.sh
```

### Execution Patterns (`30-patterns.rules`)

`fapolicyd` can match how a binary is invoked using `pattern=`:

| Pattern | Meaning |
|---|---|
| `normal` | Normal execution of a dynamically linked binary. |
| `static` | Execution of a statically linked binary. |
| `ld_preload` | Execution with `LD_PRELOAD` set in the environment. |
| `ld_so` | Direct execution of a binary via the dynamic linker (for example, `/usr/lib64/ld-linux-x86-64.so.2 /path/to/bin`). |

{: .warning }
> Keep `deny_audit perm=any pattern=ld_so : all` enabled in `/etc/fapolicyd/rules.d/30-patterns.rules`. Without it, an untrusted binary can be executed indirectly through `/usr/lib64/ld-linux-x86-64.so.2`, bypassing standard execution checks. (If container runtimes such as `runc` require `ld_so`, scope an allow rule specifically to `exe=/usr/bin/runc`.)

### File-Specific Allow Rule

```text
allow perm=execute exe=/usr/bin/bash trust=1 : path=/opt/myapp/bin/task.sh ftype=text/x-shellscript trust=0
```

Use this when a trusted interpreter or parent process must execute one specific file.

### Source-Restricted Directory Rule

```text
allow perm=execute exe=/usr/bin/bash trust=1 : dir=/opt/myapp/scripts/ trust=0
```

The trailing slash makes the directory match recursive.

### SHA-256 Hash Rule

Calculate the file checksum:

```bash
sha256sum /opt/myapp/bin/tool
```

Create a rule using the approved hash:

```text
allow perm=execute exe=/usr/bin/bash trust=1 : sha256hash=<sha256-value>
```

### Broad Directory Rule

```text
allow perm=execute all : dir=/application/dynamic-content/
```

{: .warning }
> A broad directory rule allows execution of content anywhere beneath that directory. Do not use this pattern for `/tmp`, Downloads directories, user-writable paths, removable media, or other untrusted-content locations. Use source executable, user, group, path, file type, or trust conditions whenever practical.

## Custom Rule Workflow

1. Identify the denied action.
2. Review audit records and active rules.
3. Create a narrowly scoped local rule.
4. Validate and compile the rule set.
5. Reload or restart `fapolicyd`.
6. Re-test the application.
7. Return to enforcing mode after validation.

```bash
# Review recent fapolicyd denials
ausearch -ts recent -m fanotify -i

# Review the source rule files
ls -1 /etc/fapolicyd/rules.d/

# Review the active compiled rules
fapolicyd-cli --list

# Create or edit a local rule file
vi /etc/fapolicyd/rules.d/80-local.rules

# Validate source rules
fagenrules --check

# Compile source rules into /etc/fapolicyd/compiled.rules
fagenrules --load

# Reload the compiled rules
fapolicyd-cli --reload-rules

# Restart the service if a reload is unavailable or fails
systemctl restart fapolicyd

# Verify the active rule order
fapolicyd-cli --list
```

## Troubleshooting Denials

Before creating an exception, identify what was denied and why.

```bash
# Review recent fapolicyd audit records
ausearch -ts recent -m fanotify -i

# Review fapolicyd messages from the current boot
journalctl -u fapolicyd -b

# Review service status
systemctl status fapolicyd

# Validate daemon configuration
fapolicyd-cli --check-config

# Validate trust database
fapolicyd-cli --check-trustdb

# List active rules in order
fapolicyd-cli --list
```

Look for values similar to:

```text
dec=deny_audit
perm=execute
path=/opt/myapp/bin/tool
exe=/usr/bin/bash
trust=0
rule=<number>
```

| Field | Question to Answer |
|---|---|
| `dec=` | Was the action denied or allowed? |
| `perm=` | Was the denied action execution, opening, or another operation? |
| `path=` | Which file or directory was affected? |
| `exe=` | Which executable attempted the action? |
| `uid=` / `auid=` | Which account initiated the action? |
| `trust=` | Was the target considered trusted? |
| `rule=` | Which compiled rule produced the decision? |

### Debug Modes (`--debug-deny` and `--debug`)

Use standalone debug mode only when normal audit records do not provide enough detail. Use `--debug-deny` to record only denied events, or `--debug` to record both allow and deny events.

```bash
# Stop the normal service before running standalone debug mode
systemctl stop fapolicyd

# Run debug mode and record denial information
fapolicyd --debug-deny 2> /root/fapolicyd-deny.log &
```

Reproduce the issue, then review the output:

```bash
grep -E 'deny_audit|deny' /root/fapolicyd-deny.log
```

Stop debug mode and restore the service:

```bash
pkill -f 'fapolicyd --debug-deny'
systemctl start fapolicyd
```

### Fapolicyd Analyzer (GUI)

For graphical environments, `fapolicy-analyzer` provides a GUI to inspect the System Trust Database and Ancillary Trust Database, manage trust entries, and analyze audit/syslog events.

## Integrity Checking

By default (`integrity = none`), `fapolicyd` does not verify file contents. Without integrity checking, altering or replacing an already-trusted binary on disk (for example, overwriting an RPM-managed executable) will not be detected.

Available integrity methods include:

| Value | Description |
|---|---|
| `none` | No additional integrity checking. |
| `size` | Compares file size; fast but less secure. |
| `sha256` | Compares SHA-256 checksums; more secure but has performance cost. |
| `ima` | Uses the kernel Integrity Measurement Architecture; requires additional IMA configuration and suitable filesystem support. |

To enable SHA-256 integrity checking:

```ini
# /etc/fapolicyd/fapolicyd.conf
integrity = sha256
```

```bash
systemctl restart fapolicyd
```

{: .warning }
> Test integrity checking in a non-production environment or permissive mode first. SHA-256 integrity checking can affect system performance, especially on systems with many executable files or frequent application updates.

## RPM-Managed Applications

Applications installed through DNF are normally added to the RPM database and trusted automatically.

```bash
# Install an application through DNF
dnf install <package-name>
```

If installing an RPM directly with `rpm -i`, refresh the fapolicyd database afterward:

```bash
# Install a local RPM
rpm -i <application>.rpm

# Refresh fapolicyd trust data
fapolicyd-cli --update
```

## Automating with RHEL System Roles

In multi-node environments, `fapolicyd` can be deployed and configured using the `redhat.rhel_system_roles.fapolicyd` Ansible role:

```yaml
- name: Configure fapolicyd
  hosts: all
  roles:
    - role: redhat.rhel_system_roles.fapolicyd
      vars:
        fapolicyd_setup_permissive: false
        fapolicyd_setup_integrity: sha256
        fapolicyd_setup_trust: rpmdb,file
```

## Reviewing Existing Configuration

```bash
# Check service status
systemctl status fapolicyd

# Review daemon configuration
cat /etc/fapolicyd/fapolicyd.conf

# Validate daemon configuration
fapolicyd-cli --check-config

# List source rule files in lexical order
ls -1 /etc/fapolicyd/rules.d/

# Review all source rules
grep -R -n \
  --exclude='*.rpmnew' \
  --exclude='*.rpmsave' \
  . /etc/fapolicyd/rules.d/

# Review compiled rules
cat /etc/fapolicyd/compiled.rules

# Review active rules and their order
fapolicyd-cli --list

# Review trust files
cat /etc/fapolicyd/fapolicyd.trust
find /etc/fapolicyd/trust.d/ -maxdepth 1 -type f \
  -print -exec cat {} \;

# Validate and dump the active trust database
fapolicyd-cli --check-trustdb
fapolicyd-cli --dump-db
```

## fapolicyd Quick Reference

| Task | Command |
|---|---|
| Install fapolicyd | `dnf install -y fapolicyd` |
| Enable and start fapolicyd | `systemctl enable --now fapolicyd` |
| Check service status | `systemctl status fapolicyd` |
| Review recent denials | `ausearch -ts recent -m fanotify -i` |
| Review active rules | `fapolicyd-cli --list` |
| Validate daemon configuration | `fapolicyd-cli --check-config` |
| Validate trust database | `fapolicyd-cli --check-trustdb` |
| Dump active trust database | `fapolicyd-cli --dump-db` |
| Add a trusted file | `fapolicyd-cli --file add /path/to/file --trust-file <name>` |
| Add an existing directory tree to trust | `fapolicyd-cli --file add /path/to/directory/ --trust-file <name>` |
| Remove a trusted file | `fapolicyd-cli --file delete /path/to/file --trust-file <name>` |
| Refresh one changed trust entry | `fapolicyd-cli --file update /path/to/file` |
| Refresh all trust entries | `fapolicyd-cli --file update` |
| Update active trust database | `fapolicyd-cli --update` |
| Inspect fapolicyd MIME type | `fapolicyd-cli --ftype /path/to/file` |
| Run in debug-deny mode | `fapolicyd --debug-deny` |
| Run in full debug mode | `fapolicyd --debug` |
| Validate source rules | `fagenrules --check` |
| Compile rule files | `fagenrules --load` |
| Reload compiled rules | `fapolicyd-cli --reload-rules` |
| Restart fapolicyd | `systemctl restart fapolicyd` |
| Enable permissive mode | Set `permissive = 1` in `/etc/fapolicyd/fapolicyd.conf` |
| Enable SHA-256 integrity checking | Set `integrity = sha256` in `/etc/fapolicyd/fapolicyd.conf` |
