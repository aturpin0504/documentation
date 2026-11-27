---
layout: default
title: fapolicyd
parent: Quick Reference
grand_parent: RHEL
nav_order: 19
---

# fapolicyd: File Access Policy Daemon (Admin Guide)

## What `fapolicyd` Is

`fapolicyd` is an application allow-listing daemon for RHEL that controls which programs can execute and which files processes can open.

At runtime, `fapolicyd` consumes kernel `fanotify` events: when a process attempts to open or execute a file, the kernel pauses the operation and asks `fapolicyd` for a decision. `fapolicyd` evaluates the request against its **trust database** (`RPMDB` and local `File` trust lists) and its **compiled ruleset**, then returns an allow or deny decision.

- **RPM-managed software** installed via `dnf` is trusted automatically. Installing `fapolicyd` also installs `rpm-plugin-fapolicyd` (which updates trust when packages change) and `fapolicyd-selinux`.
- **Non-RPM software** (custom scripts, third-party binaries, web apps) is untrusted by default unless you add it to the trust database or allow it with a rule.

{: .note }
> This guide covers the `rules.d/`, `trust.d/`, `fagenrules`, and `fapolicyd-cli` workflow used in RHEL 8.6–10.

---

## Configuration Files (Generated vs. Manual)

When working as `root` (for example, after `sudo -i`), some files are managed by CLI tools while others are edited manually:

| Path | Purpose | How It Is Managed |
|---|---|---|
| `/etc/fapolicyd/fapolicyd.conf` | Daemon settings (`permissive`, `integrity`, `syslog_format`). | **Manual edit in place** (`vi`). |
| `/etc/fapolicyd/fapolicyd.trust` | Default list of custom trusted files. | **Command-generated** (`fapolicyd-cli --file ...`). |
| `/etc/fapolicyd/trust.d/` | Named custom trust files (one per app). | **Command-generated** (`fapolicyd-cli --file ...`). |
| `/var/lib/fapolicyd/` | Runtime LMDB trust database. | **Command-generated** (`fapolicyd-cli --update`). Never edit manually. |
| `/etc/fapolicyd/rules.d/` | Modular rule files (`NN-name.rules`). | **Manual create/edit** (`vi`). |
| `/etc/fapolicyd/compiled.rules` | Active ruleset merged from `rules.d/`. | **Command-generated** (`fagenrules --load`). Never edit manually. |

### Ownership, Permissions, and SELinux Context

- **Command-generated files and in-place edits**: `fapolicyd-cli` and `fagenrules` set the correct ownership, permissions, and SELinux labels automatically. Editing `fapolicyd.conf` in place preserves its existing metadata.
- **Manually creating `.rules` files**: Create new `.rules` files directly inside `/etc/fapolicyd/rules.d/` so they inherit the directory's SELinux context.
  - If you ever `mv` a file from `/tmp` or `/root`, restore its SELinux label with `restorecon -v /etc/fapolicyd/rules.d/<file>.rules`.
  - If your `root` account uses a hardened `umask` (such as `0077`), ensure the file is readable (`chmod 644`, or `chown root:fapolicyd && chmod 640` to match shipped rules).

---

## How Rules Work (First Match Wins)

{: .important }
> Rule files in `/etc/fapolicyd/rules.d/` are merged in **numerical filename order**, and `fapolicyd` evaluates them from top to bottom using **first-match-wins**. As soon as one rule matches a request, evaluation stops.

Because of first-match-wins, **an `allow` rule must appear before the `deny` rule it is meant to override**.

Typical default rule prefixes:

| Prefix | Purpose |
|---|---|
| `10`–`21` | Language, dracut, and updater rules |
| `30` | Execution pattern rules (`30-patterns.rules`) |
| `40`–`42` | ELF and shared-library rules |
| `70`–`72` | Trusted language (`%languages`, including PHP/Python) and shell rules |
| `90` | Catch-all deny-execution rules (`90-deny-execute.rules`) |
| `95` | Catch-all allow-open rules (`95-allow-open.rules`) |

- To allow an **interpreted script** (such as PHP or Python) that would otherwise be blocked by `70-trusted-lang.rules`, use an earlier prefix like `50-myapp.rules`.
- To allow an **executable** that would otherwise be blocked by `90-deny-execute.rules`, use an earlier prefix like `80-local.rules`.

```bash
# View source rule files in processing order
ls -1 /etc/fapolicyd/rules.d/

# View the active compiled rules with rule numbers
fapolicyd-cli --list
```

{: .warning }
> Keep `deny_audit perm=any pattern=ld_so : all` enabled in `/etc/fapolicyd/rules.d/30-patterns.rules`. Without it, an untrusted binary can be run indirectly through `/usr/lib64/ld-linux-x86-64.so.2`, bypassing standard execution checks.

---

## What Rules Can Match On

Every rule uses Subject/Object notation:

```text
decision permission subject : object
```

- **Subject** (left of `:`): the process or user **making** the request (for example, `php-fpm` or `bash`).
- **Object** (right of `:`): the target file **being accessed** (opened or executed).

### Decisions and Permissions

| Field | Values | Meaning |
|---|---|---|
| **Decision** | `allow`, `allow_syslog`, `allow_audit` | Permit the action (optionally logging to syslog or audit). |
| | `deny`, `deny_syslog`, `deny_audit` | Block the action (optionally logging to syslog or audit). |
| **Permission** | `perm=open` | Matches file-open requests (libraries, scripts read by an interpreter). |
| | `perm=execute` | Matches execution attempts. |
| | `perm=any` | Matches both `open` and `execute`. |

### Subject Attributes (Left of `:`)

| Attribute | Meaning | Example |
|---|---|---|
| `all` | Matches any process and any user. | `all` |
| `exe=` | Full path of the running executable making the request. | `exe=/usr/sbin/php-fpm` |
| `comm=` | Short command name of the process. | `comm=bash` |
| `uid=` / `gid=` | User ID or group ID of the process. | `uid=48` |
| `auid=` | Login (audit) user ID of the session. | `auid=1000` |
| `trust=` | Whether the **calling executable itself** is trusted (`1`) or untrusted (`0`). | `trust=1` |
| `pattern=` | Execution pattern (`normal`, `static`, `ld_preload`, `ld_so`). | `pattern=ld_so` |

#### Understanding `exe=` vs. `trust=1` on the Subject Side

A common point of confusion is why a rule includes **both** `exe=/usr/bin/bash` and `trust=1` on the subject side (for example, `allow perm=execute exe=/usr/bin/bash trust=1 : dir=/opt/myapp/scripts/ trust=0`).

- **`exe=`** only checks **the path** of the program making the request. It does not check whether that program is in the trust database.
- **`trust=1` (subject side)** verifies that the calling program (`exe=`) is **itself in the `fapolicyd` trust database** (and passes integrity checks if enabled).

This rule does not mean "allow `bash` to run"—`bash` is already running and trying to open or execute a file in `/opt/myapp/scripts/`. Together, `exe=/usr/bin/bash trust=1` means: *"Only apply this allow rule if the caller is `/usr/bin/bash` **and** `/usr/bin/bash` itself is a trusted binary."* Meanwhile, `trust=0` on the object side describes the target script, which is not in the trust database.

### Object Attributes (Right of `:`)

| Attribute | Meaning | Example |
|---|---|---|
| `all` | Matches any target file. | `all` |
| `path=` | Exact path to the target file. | `path=/opt/myapp/bin/tool` |
| `dir=` | Target directory. A trailing `/` matches subdirectories recursively. | `dir=/var/www/html/password/` |
| `ftype=` | File MIME type as `fapolicyd` sees it (verify with `fapolicyd-cli --ftype`). | `ftype=text/x-php` |
| `sha256hash=` | Exact SHA-256 checksum of the target file. | `sha256hash=<64-char-hex>` |
| `trust=` | Whether the target file is in the trust database (`1`) or not (`0`). | `trust=0` |

Before writing `ftype=` rules, check the exact MIME type `fapolicyd` assigns to a file (it can differ from `file`):

```bash
fapolicyd-cli --ftype /var/www/html/password/index.php
```

---

## How to Allow Applications

### Method 1: Trust Static Files (`fapolicyd-cli`)

Best for non-RPM binaries or scripts whose contents rarely change. Every entry records the file's size and SHA-256 hash.

```bash
# Add a single file or directory to a named trust file (/etc/fapolicyd/trust.d/myapp)
fapolicyd-cli --file add /opt/myapp/bin/tool --trust-file myapp
fapolicyd-cli --file add /opt/myapp/ --trust-file myapp

# Refresh trust entries after a file changes
fapolicyd-cli --file update /opt/myapp/bin/tool

# Apply changes to the active trust database
fapolicyd-cli --update
```

{: .warning }
> Adding a directory with `--file add` only trusts the files present at that moment. New or modified files require `fapolicyd-cli --file update` and `fapolicyd-cli --update`. Avoid trusting directories with thousands of frequently changing files (such as PHP cache/session folders)—use a scoped `dir=` rule instead.

### Method 2: Define Custom Rules (`rules.d/`)

Best when you need to allow a directory regardless of checksum changes, restrict access to a specific calling program (`exe=`), or match a specific SHA-256 hash.

1. Create a rule file in `/etc/fapolicyd/rules.d/` with an appropriate numerical prefix:
   ```bash
   vi /etc/fapolicyd/rules.d/50-myapp.rules
   ```
2. Add your rule(s):
   - **Allow a specific script** from a trusted interpreter:
     ```text
     allow perm=execute exe=/usr/bin/bash trust=1 : path=/opt/myapp/bin/task.sh ftype=text/x-shellscript trust=0
     ```
   - **Allow a dynamic directory** (such as a PHP web app) regardless of file checksums:
     ```text
     allow perm=open exe=/usr/sbin/php-fpm trust=1 : dir=/var/www/html/password/ ftype=text/x-php
     ```
   - **Allow a specific file by SHA-256 hash**:
     ```text
     allow perm=execute exe=/usr/bin/bash trust=1 : sha256hash=<sha256-value>
     ```
3. Validate, compile, and reload:
   ```bash
   fagenrules --check
   fagenrules --load
   fapolicyd-cli --reload-rules
   ```

---

## Testing, Integrity, and Troubleshooting

### Safe Rollout with Permissive Mode

Enable permissive mode in `/etc/fapolicyd/fapolicyd.conf` when testing new apps or rules so `fapolicyd` logs denials without blocking them:

```ini
# /etc/fapolicyd/fapolicyd.conf
permissive = 1
```

```bash
systemctl restart fapolicyd
```

Set `permissive = 0` and restart `fapolicyd` once verified.

### Finding What Is Being Blocked

Query recent `fanotify` audit events without stopping the service:

```bash
ausearch -ts recent -m fanotify -i
```

Or run `fapolicyd` briefly in `--debug-deny` mode to log only denials in real time:

```bash
systemctl stop fapolicyd
fapolicyd --debug-deny 2> /root/fapolicyd-deny.log &
# Reproduce the issue, then stop debug mode and restart the service:
pkill -f 'fapolicyd --debug-deny'
systemctl start fapolicyd
grep -E 'deny_audit|deny' /root/fapolicyd-deny.log
```

A denial line looks like:

```text
rule=11 dec=deny_audit perm=open auid=48 pid=10245 exe=/usr/sbin/php-fpm : path=/var/www/html/password/index.php ftype=text/x-php trust=0
```

- `rule=11`: Which rule in `fapolicyd-cli --list` blocked the action (your new allow rule must have a lower number).
- `perm=`, `exe=`, `path=`, `ftype=`: The exact values needed to trust the file or write a targeted allow rule.

### Removing Files from `fapolicyd`

If files are deleted, moved, or replaced, `fapolicyd-cli --check-trustdb` will report missing files.

```bash
# Remove a single file from a named trust file
fapolicyd-cli --file delete /opt/myapp/bin/tool --trust-file myapp
fapolicyd-cli --update

# Remove an entire named trust file (for example, to replace a large trust list with a dir= rule)
rm /etc/fapolicyd/trust.d/teampass
fapolicyd-cli --update
fapolicyd-cli --check-trustdb
```

### Integrity Checking

By default (`integrity = none`), `fapolicyd` only checks whether a file path is in the trust database—it does not verify file contents. If a trusted binary is altered on disk, `fapolicyd` will not detect it unless integrity checking is enabled in `/etc/fapolicyd/fapolicyd.conf`:

| Value | Behavior |
|---|---|
| `none` | No content verification (default). |
| `size` | Compares file size; fast, but does not detect same-size tampering. |
| `sha256` | Verifies SHA-256 checksums; strong security, higher CPU/I/O cost. |
| `ima` | Uses the kernel Integrity Measurement Architecture (requires IMA setup). |

---

## Quick Reference

| Task | Command |
|---|---|
| Install `fapolicyd` | `dnf install -y fapolicyd` |
| Enable and start service | `systemctl enable --now fapolicyd` |
| Check service status | `systemctl status fapolicyd` |
| Find recent blocks | `ausearch -ts recent -m fanotify -i` |
| Run standalone deny debugger | `fapolicyd --debug-deny` |
| List active rules in order | `fapolicyd-cli --list` |
| Check `fapolicyd` MIME type | `fapolicyd-cli --ftype /path/to/file` |
| Add file/dir to named trust | `fapolicyd-cli --file add /path/ --trust-file <name>` |
| Delete file from named trust | `fapolicyd-cli --file delete /path/to/file --trust-file <name>` |
| Refresh changed trust entries | `fapolicyd-cli --file update` |
| Apply trust changes to DB | `fapolicyd-cli --update` |
| Validate trust database | `fapolicyd-cli --check-trustdb` |
| Validate config file | `fapolicyd-cli --check-config` |
| Check and compile rules | `fagenrules --check && fagenrules --load` |
| Reload compiled rules | `fapolicyd-cli --reload-rules` |
