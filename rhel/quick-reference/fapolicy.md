---
layout: default
title: "fapolicyd: File Access Policy Daemon"
parent: Quick Reference
grand_parent: RHEL
nav_order: 18
---

# fapolicyd: File Access Policy Daemon

## Overview

`fapolicyd` is an application allow-listing framework that restricts execution of untrusted applications. Administrators can define allow, deny, and audit rules based on attributes such as path, hash, MIME type, and trust status. 

### Trust Concept

Applications are generally trusted when they are properly installed through the system package manager and registered in the RPM database. `fapolicyd` uses the RPM database as a source of trusted binaries and scripts, while the `fapolicyd` RPM plugin tracks DNF/RPM-managed updates and notifies the daemon of relevant database changes. 

{: .note }
> When RHEL is deployed using a DISA security profile, verify whether `fapolicyd` has already been installed and configured by the applicable baseline or automation before making changes.

## Configuration Files

| File or Directory | Purpose |
|---|---|
| `/etc/fapolicyd/fapolicyd.conf` | Main `fapolicyd` configuration file. |
| `/etc/fapolicyd/fapolicyd.trust` | Default list of trusted files. |
| `/etc/fapolicyd/trust.d/` | Additional trusted-file lists. |
| `/etc/fapolicyd/rules.d/` | Component files containing allow and deny execution rules. |
| `/etc/fapolicyd/compiled.rules` | Compiled rules generated from files in `rules.d/`. |

The `fagenrules` utility merges rule files from `/etc/fapolicyd/rules.d/` into `/etc/fapolicyd/compiled.rules`. 

## Rule Order

{: .important }
> Rule files in `/etc/fapolicyd/rules.d/` are processed in numerical filename order. Rules are evaluated in order, so custom allow rules must appear before a matching deny rule.

Example order:

```text
80-myapps.rules
90-deny-execute.rules
```

In this example, rules in `80-myapps.rules` are evaluated before rules in `90-deny-execute.rules`.

```bash
# Verify compiled rule order
fapolicyd-cli --list
```

Red Hat documents the use of numbered rule files, including a `90` deny-execute rule set; the leading numbers determine the resulting order in `compiled.rules`. 

## Marking Files as Trusted

| Task | Command |
|---|---|
| Mark a single file as trusted in a named trust file | `fapolicyd-cli --file add /tmp/ls --trust-file myapp` |
| Mark directory contents as trusted in a named trust file | `fapolicyd-cli --file add /tmp/my_bin_dir/ --trust-file myapp` |
| Add a file to the default trust file | `fapolicyd-cli --file add /path/to/file` |
| Update the trust database | `fapolicyd-cli --update` |
| Refresh checksums for a file | `fapolicyd-cli --file update /path/to/file` |
| Refresh the entire file trust database | `fapolicyd-cli --file update` |

### Using `--trust-file`

The `--trust-file` parameter specifies the file to create or update under `/etc/fapolicyd/trust.d/`.

Example:

```bash
fapolicyd-cli --file add /tmp/ls --trust-file myapp
```

This creates or updates:

```text
/etc/fapolicyd/trust.d/myapp
```

If `--trust-file` is omitted, entries are added to:

```text
/etc/fapolicyd/fapolicyd.trust
```

Use separate trust files to organize entries by application or function:

```bash
fapolicyd-cli --file add /opt/satellite/bin/tool --trust-file satellite
```

RHEL supports the default `fapolicyd.trust` file and additional trust files under `/etc/fapolicyd/trust.d/`. 

### Refreshing Trust After File Changes

Changing a trusted file changes its checksum. Refresh the file and trust databases after modifying trusted content:

```bash
# Refresh checksum data for the changed file
fapolicyd-cli --file update /path/to/file

# Update the trust database
fapolicyd-cli --update
```

### Trust Marking and Dynamic Content

Trust marking applies to files that exist when they are added to the trust database. For directories where content is created dynamically, such as an application working directory, use custom rules instead of relying solely on checksum-based trust entries.

For example, an application directory with dynamic content may require a rule such as:

```text
allow perm=execute all : dir=/var/satellite/
```

## Enabling `fapolicyd`

### Enable Permissive Mode for Testing

Edit the main configuration file:

```bash
vi /etc/fapolicyd/fapolicyd.conf
```

Set:

```ini
permissive = 1
```

Enable and start the service:

```bash
systemctl enable --now fapolicyd
```

Check service status:

```bash
systemctl status fapolicyd
```

Search recent audit records for `fapolicyd` denials:

```bash
ausearch -ts recent -m fanotify
```

### Enable Enforcing Mode

After testing and correcting policy issues, change the setting back to:

```ini
permissive = 0
```

Restart the service:

```bash
systemctl restart fapolicyd
```

Red Hat recommends testing configuration in permissive mode, checking audit logs for `fanotify` denials, then returning to enforcing behavior after troubleshooting. 

## Adding Custom Rules

### Recommended Workflow

1. Enable permissive mode in `/etc/fapolicyd/fapolicyd.conf`:

   ```ini
   permissive = 1
   ```

2. Restart the service:

   ```bash
   systemctl restart fapolicyd
   ```

3. Reproduce the denied command or application execution:

   ```bash
   /tmp/ls
   ```

4. Review recent denials:

   ```bash
   ausearch -ts recent -m fanotify
   ```

5. Review current rule files:

   ```bash
   ls /etc/fapolicyd/rules.d/
   ```

6. Create a custom allow-rule file that precedes deny rules:

   ```bash
   vi /etc/fapolicyd/rules.d/80-myapps.rules
   ```

7. Check rule syntax:

   ```bash
   fagenrules --check
   ```

8. Compile and load the rules:

   ```bash
   fagenrules --load
   ```

9. Verify rule order:

   ```bash
   fapolicyd-cli --list
   ```

10. Disable permissive mode:

    ```ini
    permissive = 0
    ```

11. Restart the service:

    ```bash
    systemctl restart fapolicyd
    ```

12. Test the application again:

    ```bash
    /tmp/ls
    ```

Red Hat documents custom rules in a file such as `80-myapps.rules`, followed by `fagenrules --check` and `fagenrules --load` to validate and apply the configuration. 

## Rule Structure

Rules use the following general format:

```text
SOURCE : TARGET
```

### Trust Values

| Value | Meaning |
|---|---|
| `trust=1` | Object must be trusted, such as through the RPM database or configured trust source. |
| `trust=0` | Object does not need to be present in the RPM database or trust database. |

## Rule Examples

### Allow a Specific File

```text
allow perm=execute exe=/usr/bin/bash trust=1 : path=/tmp/ls ftype=application/x-executable trust=0
```

This permits trusted `/usr/bin/bash` to execute the untrusted executable `/tmp/ls`.

### Allow a Directory Recursively

```text
allow perm=execute exe=/usr/bin/bash trust=1 : dir=/tmp/ trust=0
```

The trailing slash in `dir=/tmp/` applies the rule recursively to the directory and its subdirectories.

### Allow Execution from an Application Directory

```text
allow perm=execute all : dir=/var/satellite/
```

This permits any process to execute files under `/var/satellite/` without requiring trust validation for those target files.

{: .warning }
> The simple `all : dir=/path/` rule is broad. Use more restrictive rules with source executable and trust conditions for sensitive paths whenever practical.

## Choosing a Rule Type

| Rule Type | Example | Appropriate Use |
|---|---|---|
| Simple directory allow rule | `allow perm=execute all : dir=/path/` | Application directories containing dynamic content, such as `/var/satellite/`. |
| Restrictive allow rule | `allow perm=execute exe=/usr/bin/bash trust=1 : dir=/path/ trust=0` | Sensitive paths where only a specific trusted process should execute untrusted content. |
| File-specific allow rule | `allow perm=execute exe=/usr/bin/bash trust=1 : path=/tmp/ls ftype=application/x-executable trust=0` | Individual binaries or scripts requiring explicit exception handling. |

## Rule File Management

| Task | Command |
|---|---|
| List all rules in processing order | `fapolicyd-cli --list` |
| Check rule syntax | `fagenrules --check` |
| Compile and load rules | `fagenrules --load` |
| View compiled rules | `cat /etc/fapolicyd/compiled.rules` |

## Best Practices

- Prefer marking static files as trusted over creating custom allow rules when possible.
- Use custom rules for directories containing dynamically created files.
- Number custom rule files to control evaluation order.
- Place allow rules before applicable deny rules—for example, use `80-myapps.rules` before `90-deny-execute.rules`.
- Run `fagenrules --check` before `fagenrules --load`.
- Test changes in permissive mode before enforcing them.
- Monitor recent denials with:

  ```bash
  ausearch -ts recent -m fanotify
  ```

- Verify final compiled-rule order with:

  ```bash
  fapolicyd-cli --list
  ```