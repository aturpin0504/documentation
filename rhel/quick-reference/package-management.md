---
layout: default
title: "Package & Satellite Management"
parent: Quick Reference
grand_parent: RHEL
---

# Package & Satellite Management

Daily package management (`dnf`, `rpm`), module streams, custom STIG-compliant repos, and Red Hat Satellite 6.19 maintenance.

---

## Satellite Server Registration & Manifests

If a Satellite 6.19 server (running on RHEL 9) needs to be rebuilt, register it to Red Hat CDN and enable only the required repositories:

```bash
# 1. Register the host to Red Hat
subscription-manager register --username <rh_user> --auto-attach

# 2. Disable all repos first to avoid conflicts
subscription-manager repos --disable "*"

# 3. Enable the required RHEL 9 + Satellite 6.19 repos
subscription-manager repos \
  --enable=rhel-9-for-x86_64-baseos-rpms \
  --enable=rhel-9-for-x86_64-appstream-rpms \
  --enable=satellite-6.19-for-rhel-9-x86_64-rpms \
  --enable=satellite-maintenance-6.19-for-rhel-9-x86_64-rpms
```

### Subscription Manifests

A **Subscription Manifest** (`.zip`) carries your Red Hat entitlements and CDN certificates from the Red Hat Hybrid Cloud Console into Satellite so Satellite can sync Red Hat repositories.

1. **Export:** In [console.redhat.com](https://console.redhat.com), go to **Subscriptions → Allocations**, select your Satellite allocation, and click **Export Manifest** to download the `.zip` file.
2. **Import into Satellite:**
   - **Web UI:** **Content → Subscriptions → Manage Manifest → Upload**.
   - **CLI (`hammer`):**
     ```bash
     hammer subscription upload --file ~/manifest.zip --organization "Your_Org"
     ```
3. **Refresh later (after adding/renewing subscriptions):**
   ```bash
   hammer subscription refresh-manifest --organization "Your_Org"
   ```

---

## Repolist Management on Hosts

```bash
dnf repolist                                   # List enabled repositories
dnf repolist all                               # List both enabled and disabled repositories
subscription-manager repos --list-enabled      # List enabled Red Hat/Satellite repos
subscription-manager repos --enable=<repo-id>  # Enable a repo via subscription-manager
subscription-manager repos --disable=<repo-id> # Disable a repo via subscription-manager
dnf config-manager --enable <repo-id>          # Enable a repo in /etc/yum.repos.d/
dnf config-manager --disable <repo-id>         # Disable a repo in /etc/yum.repos.d/
```

---

## Custom Repositories & STIG GPG Signing

### Add, Modify, or Remove a Custom Repo

- **Add a repo:**
  ```bash
  dnf config-manager --add-repo https://example.com/custom/custom.repo
  ```
  Or create `/etc/yum.repos.d/custom.repo` manually:
  ```ini
  [custom-apps]
  name=Custom Internal Apps
  baseurl=https://repo.example.com/custom-apps/el9/x86_64/
  enabled=1
  gpgcheck=1
  gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-custom
  ```
- **Modify a repo:** Edit `/etc/yum.repos.d/<name>.repo` directly, or use `dnf config-manager`:
  ```bash
  dnf config-manager --setopt=custom-apps.enabled=0 --save
  ```
- **Remove an accidentally created repo:**
  ```bash
  rm /etc/yum.repos.d/<name>.repo
  dnf clean all
  ```

### STIG-Compliant GPG Keys & Signing Custom Packages

> [!WARNING]
> **FIPS & STIG Key Type Requirement**
> RHEL in FIPS mode and DISA STIG **do not allow Ed25519 (EdDSA)** keys (`ED25519 keys are not allowed in FIPS mode`). Use **RSA (4096-bit or 2048-bit)** for GPG package-signing keys.

1. **Generate an RSA-4096 GPG key:**
   ```bash
   gpg --full-generate-key
   # Choose: (1) RSA and RSA -> Keysize: 4096 -> Expiration -> Name/Email
   ```
2. **Export the public key:**
   ```bash
   gpg --armor --export "admin@example.com" > RPM-GPG-KEY-custom
   ```
3. **Configure `rpm` to use your key (`~/.rpmmacros`):**
   ```text
   %_gpg_name admin@example.com
   ```
4. **Sign custom RPMs before uploading to Satellite or a local repo:**
   ```bash
   rpmsign --addsign /path/to/my-package-1.0-1.el9.x86_64.rpm
   rpm -K /path/to/my-package-1.0-1.el9.x86_64.rpm   # Verify signature
   ```
5. **Trust the public key on hosts:**
   - **In Satellite:** Upload `RPM-GPG-KEY-custom` under **Content → Content Credentials** and attach it to your custom Product/Repository.
   - **Directly on a host:**
     ```bash
     cp RPM-GPG-KEY-custom /etc/pki/rpm-gpg/
     rpm --import /etc/pki/rpm-gpg/RPM-GPG-KEY-custom
     ```

*(If hosting a standalone repo outside Satellite, generate repo metadata with `createrepo_c /path/to/repo`.)*

---

## Package Management (`dnf` & `satellite-maintain`)

### Standard Host (`dnf`)

```bash
dnf install <pkg>        # Install a package
dnf remove <pkg>         # Remove a package
dnf check-update         # Check for available updates
dnf update               # Update all packages
dnf update <pkg>         # Update a specific package
dnf clean all            # Clear cached packages and metadata
```

### Satellite Server Updates (`satellite-maintain`)

Direct `dnf update` is locked on Satellite servers to protect Satellite's dependencies. Always use `satellite-maintain`:

```bash
satellite-maintain health check      # Run pre-update health checks
satellite-maintain packages update   # Update Satellite OS & maintenance packages safely
```

> [!IMPORTANT]
> **Required Repos for `satellite-maintain packages update` (Satellite 6.19)**
> If `satellite-maintain packages update` fails with missing repos or dependency errors, verify all four of these repositories are enabled on the Satellite server:
> - `rhel-9-for-x86_64-baseos-rpms`
> - `rhel-9-for-x86_64-appstream-rpms`
> - `satellite-6.19-for-rhel-9-x86_64-rpms`
> - `satellite-maintenance-6.19-for-rhel-9-x86_64-rpms`

---

## `yum` / `dnf` History

`yum` is symlinked to `dnf` on RHEL 8/9, so `yum history` and `dnf history` are identical:

```bash
dnf history                  # List past transactions with IDs
dnf history info <id>        # Inspect packages changed in a specific transaction
dnf history undo <id>        # Roll back a transaction
dnf history redo <id>        # Replay a transaction
```

---

## Module Streams (e.g., Updating PHP)

Application Streams let you switch between major versions of software like PHP.

```bash
# 1. List available streams ([d] = default, [e] = enabled, [i] = installed)
dnf module list php

# 2. Reset the currently enabled stream
dnf module reset php

# 3. Enable the desired newer stream (e.g., 8.2)
dnf module enable php:8.2

# 4. Sync/update installed packages to the new stream
dnf distro-sync
```

*(If PHP isn't installed yet, run `dnf install @php:8.2` after enabling the stream.)*

---

## Searching for Packages

```bash
dnf search <keyword>             # Search package names and descriptions
dnf provides */<filename>        # Find which package provides a binary/file (e.g., */semanage)
dnf info <pkg>                   # Show version, repo, size, and summary
dnf list available <pattern>     # List matching packages available in enabled repos
dnf list installed <pattern>     # List matching installed packages
rpm -qa | grep <pattern>         # Fast local search of installed RPMs
rpm -ql <pkg>                    # List all files installed by a package
```