---
layout: default
title: fapolicyd
parent: Quick Reference
grand_parent: RHEL
nav_order: 19
---

# fapolicyd: File Access Policy Daemon for RHEL 8.6–10

## Overview

`fapolicyd` is an application allow-listing framework that controls application execution based on trust and administrator-defined policy.

Administrators can define allow, deny, and audit rules based on attributes such as:

- File path
- SHA-256 hash
- MIME type
- Executable path
- User or group identity
- Trust status

Applications installed through DNF and registered in the RPM database are normally trusted. The `fapolicyd` RPM plugin monitors DNF and RPM package transactions and notifies `fapolicyd` when RPM database content changes.

{: .note }
> This guide uses the `rules.d/`, `trust.d/`, `fagenrules`, and modern `fapolicyd-cli` workflow available in RHEL 8.6 and later. For earlier RHEL 8 releases, review the installed `fapolicyd` documentation and configuration layout before applying these procedures.

{: .note }
> When RHEL is deployed using a DISA security profile, verify whether `fapolicyd` has already been installed and configured by the applicable baseline or automation before making changes.

## Configuration Files

| File or Directory | Purpose |
|---|---|
| `/etc/fapolicyd/fapolicyd.conf` | Main daemon configuration file. Contains options for permissive mode, integrity checking, performance, and watched filesystems. |
| `/etc/fapolicyd/fapolicyd.trust` | Default list of trusted files outside RPM management. |
| `/etc/fapolicyd/trust.d/` | Additional named trusted-file lists. |
| `/etc/fapolicyd/rules.d/` | Component files containing allow, deny, and audit rules. |
| `/etc/fapolicyd/compiled.rules` | Compiled rule set generated from `rules.d/` by `fagenrules`. |
| `/var/lib/fapolicyd/` | fapolicyd trust database location. Do not remove this directory. |

## Rule Order

{: .important }
> Rule files in `/etc/fapolicyd/rules.d/` are processed in numerical filename order. Rules are evaluated in order, so an allow rule must appear before the deny rule it is intended to override.

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

Remove the applicable entry from the trust file, then restart the service.

```bash
# Edit a named trust file
vi /etc/fapolicyd/trust.d/myapp

# Or edit the default trust file
vi /etc/fapolicyd/fapolicyd.trust

# Restart after removing trust entries
systemctl restart fapolicyd
```

{: .warning }
> Trusting a directory adds trust for the files present when the trust entry is created. Files added or changed later require a trust-database refresh. Do not use trust entries for directories containing untrusted, user-writable, or frequently changing content.

## Adding Custom Rules

Use custom rules when a trust entry is too broad or does not provide the required conditions.

### Rule Structure

Rules use this general format:

```text
decision permission source : target
```

Example:

```text
allow perm=execute exe=/usr/bin/bash trust=1 : path=/opt/myapp/bin/task.sh ftype=text/x-shellscript trust=0
```

| Element | Meaning |
|---|---|
| `allow` | Permit the requested action. |
| `deny_audit` | Deny the action and create an audit event. |
| `perm=execute` | Rule applies to execution attempts. |
| `exe=/usr/bin/bash` | The parent executable attempting the action. |
| `trust=1` | The referenced source object must be trusted. |
| `path=` | Matches a specific file path. |
| `dir=` | Matches a directory path. A trailing `/` applies recursively. |
| `ftype=` | Matches a file MIME type. |
| `sha256hash=` | Matches a file with the specified SHA-256 hash. |
| `trust=0` | Matches an object that is not trusted. |

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

### Debug-Deny Mode

Use debug mode only when normal audit records do not provide enough detail.

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

## Integrity Checking

By default, `fapolicyd` does not perform additional file-integrity verification.

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
| Refresh one changed trust entry | `fapolicyd-cli --file update /path/to/file` |
| Refresh all trust entries | `fapolicyd-cli --file update` |
| Update active trust database | `fapolicyd-cli --update` |
| Validate source rules | `fagenrules --check` |
| Compile rule files | `fagenrules --load` |
| Reload compiled rules | `fapolicyd-cli --reload-rules` |
| Restart fapolicyd | `systemctl restart fapolicyd` |
| Enable permissive mode | Set `permissive = 1` in `/etc/fapolicyd/fapolicyd.conf` |
| Enable SHA-256 integrity checking | Set `integrity = sha256` in `/etc/fapolicyd/fapolicyd.conf` |
