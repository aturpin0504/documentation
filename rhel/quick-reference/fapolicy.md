---
layout: default
title: fapolicyd
parent: Quick Reference
grand_parent: RHEL
nav_order: 19
---

# fapolicyd: File Access Policy Daemon

## Overview

`fapolicyd` is an application allow-listing framework that restricts execution of untrusted applications. Administrators can define allow, deny, and audit rules based on attributes such as path, hash, MIME type, and trust status.

Applications are generally trusted when installed through the system package manager and registered in the RPM database. The `fapolicyd` RPM plugin tracks DNF/RPM-managed updates and notifies the daemon of database changes.

{: .note }
> When RHEL is deployed using a DISA security profile, verify whether `fapolicyd` has already been installed and configured by the applicable baseline or automation before making changes.

## Configuration Files

| File or Directory | Purpose |
|---|---|
| `/etc/fapolicyd/fapolicyd.conf` | Main `fapolicyd` configuration file. |
| `/etc/fapolicyd/fapolicyd.trust` | Default list of trusted files. |
| `/etc/fapolicyd/trust.d/` | Additional trusted-file lists. |
| `/etc/fapolicyd/rules.d/` | Component files containing allow and deny execution rules. |
| `/etc/fapolicyd/compiled.rules` | Compiled rules generated from files in `rules.d/` by `fagenrules`. |

## Rule Order

{: .important }
> Rule files in `/etc/fapolicyd/rules.d/` are processed in numerical filename order. Rules are evaluated in order, so custom allow rules (e.g., `80-myapps.rules`) must appear before a matching deny rule (e.g., `90-deny-execute.rules`).

```bash
# Verify compiled rule order
fapolicyd-cli --list
```

## Marking Files as Trusted

Use `--trust-file <name>` to write entries to `/etc/fapolicyd/trust.d/<name>` instead of the default `/etc/fapolicyd/fapolicyd.trust`.

```bash
# Mark a single file or directory contents as trusted in a named trust file
fapolicyd-cli --file add /tmp/ls --trust-file myapp
fapolicyd-cli --file add /tmp/my_bin_dir/ --trust-file myapp
fapolicyd-cli --file add /opt/satellite/bin/tool --trust-file satellite

# Add a file to the default trust file
fapolicyd-cli --file add /path/to/file

# Refresh checksums for a changed file or the entire file trust database, then update
fapolicyd-cli --file update /path/to/file
fapolicyd-cli --file update
fapolicyd-cli --update
```

For directories where content is created dynamically (such as an application working directory), use custom rules instead of checksum-based trust entries (for example, `allow perm=execute all : dir=/var/satellite/`).

## Enabling `fapolicyd`

Test in permissive mode first by setting `permissive = 1` in `/etc/fapolicyd/fapolicyd.conf`:

```bash
# Enable, start, and check status
systemctl enable --now fapolicyd
systemctl status fapolicyd

# Search recent audit records for fapolicyd denials
ausearch -ts recent -m fanotify
```

After correcting policy issues, set `permissive = 0` in `/etc/fapolicyd/fapolicyd.conf` and restart:

```bash
systemctl restart fapolicyd
```

## Adding Custom Rules (Workflow)

1. Set `permissive = 1` in `/etc/fapolicyd/fapolicyd.conf` and restart `fapolicyd`.
2. Reproduce the denied execution (e.g., `/tmp/ls`), then inspect denials and existing rules:

```bash
ausearch -ts recent -m fanotify
ls /etc/fapolicyd/rules.d/
```

3. Create a numbered allow-rule file before deny rules (e.g., `/etc/fapolicyd/rules.d/80-myapps.rules`), then validate, compile, and verify:

```bash
fagenrules --check
fagenrules --load
fapolicyd-cli --list
```

4. Set `permissive = 0` in `/etc/fapolicyd/fapolicyd.conf`, restart `fapolicyd`, and re-test the application:

```bash
systemctl restart fapolicyd
```

## Rule Structure and Examples

Rules use the format `SOURCE : TARGET`.

| Value | Meaning |
|---|---|
| `trust=1` | Object must be trusted (RPM database or configured trust source). |
| `trust=0` | Object does not need to be present in the RPM database or trust database. |

| Rule Type | Example | Appropriate Use |
|---|---|---|
| File-specific allow rule | `allow perm=execute exe=/usr/bin/bash trust=1 : path=/tmp/ls ftype=application/x-executable trust=0` | Individual binaries or scripts requiring explicit exception handling. |
| Restrictive directory allow rule | `allow perm=execute exe=/usr/bin/bash trust=1 : dir=/tmp/ trust=0` | Sensitive paths where only a specific trusted process should execute untrusted content (trailing `/` applies recursively). |
| Broad directory allow rule | `allow perm=execute all : dir=/var/satellite/` | Application directories containing dynamic content. |

{: .warning }
> The simple `all : dir=/path/` rule is broad. Use more restrictive rules with source executable and trust conditions for sensitive paths whenever practical.

## Rule File Management

```bash
# List all rules in processing order
fapolicyd-cli --list

# Check rule syntax
fagenrules --check

# Compile and load rules into /etc/fapolicyd/compiled.rules
fagenrules --load

# View compiled rules
cat /etc/fapolicyd/compiled.rules
```