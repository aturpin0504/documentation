---
layout: default
title: "Package Management"
parent: Quick Reference
grand_parent: RHEL
nav_order: 7
---

# Package and Satellite Management

Daily package management with `dnf` and `rpm`, module streams, custom repositories, STIG-aligned GPG signing, and Red Hat Satellite maintenance.

{: .note }
> This guide uses `dnf` as the primary package-management command (`yum` is a symlink to `dnf` on systems where it is present).

## Satellite Server Registration and Required Repositories

If a Satellite server must be rebuilt, register it to Red Hat CDN and enable only the BaseOS, AppStream, Satellite, and Satellite Maintenance repositories matching your RHEL major version (`<rhel_ver>`, e.g., `9` or `10`) and Satellite version (`<sat_ver>`):

```bash
# 1. Register the host to Red Hat
subscription-manager register --username <rh_user> --auto-attach

# 2. Disable all repositories first to avoid conflicts
subscription-manager repos --disable "*"

# 3. Verify exact repository IDs for your RHEL and Satellite versions if needed
subscription-manager repos --list | grep -E "baseos-rpms|appstream-rpms|satellite-"

# 4. Enable required RHEL and Satellite repositories
subscription-manager repos \
  --enable=rhel-<rhel_ver>-for-x86_64-baseos-rpms \
  --enable=rhel-<rhel_ver>-for-x86_64-appstream-rpms \
  --enable=satellite-<sat_ver>-for-rhel-<rhel_ver>-x86_64-rpms \
  --enable=satellite-maintenance-<sat_ver>-for-rhel-<rhel_ver>-x86_64-rpms
```

If `satellite-maintain packages update` fails due to missing repositories or dependencies, verify all four repositories above are enabled.

## Subscription Manifests

A Subscription Manifest (`.zip`) carries Red Hat entitlements and CDN certificates from the Red Hat Hybrid Cloud Console into Satellite, allowing Satellite to synchronize Red Hat repositories.

### Export a Manifest

In the Red Hat Hybrid Cloud Console:

1. Navigate to **Subscriptions** → **Allocations**.
2. Select the Satellite allocation.
3. Select **Export Manifest** to download the `.zip` file.

### Import a Manifest into Satellite

**Web UI:**

```text
Content → Subscriptions → Manage Manifest → Upload
```

**Hammer CLI:**

```bash
# Import a manifest
hammer subscription upload --file ~/manifest.zip --organization "Your_Org"

# Refresh after adding or renewing subscriptions
hammer subscription refresh-manifest --organization "Your_Org"
```

## Repository and Subscription Management

### Satellite Server vs. Satellite-Managed Hosts

| System Role | How Repositories Are Managed | Primary Tool |
|---|---|---|
| **Satellite Server** | Registered directly to Red Hat CDN (plus any local custom `.repo` files under `/etc/yum.repos.d/`) | `subscription-manager` + `satellite-maintain` |
| **Satellite-Managed Host** | Controlled centrally in Satellite Web UI (**Content Views**, **Activation Keys**, **Repository Sets**) | Satellite UI + `subscription-manager` (status/refresh) |

On Satellite-managed hosts, `subscription-manager` automatically regenerates `/etc/yum.repos.d/redhat.repo`. Use `subscription-manager repos` for Red Hat/Satellite-managed repos, and `dnf config-manager` only for custom or third-party `.repo` files.

```bash
# Register a host to Satellite using an organization and activation key
subscription-manager register --org="<Your_Org>" --activationkey="<Your_Key>"

# Force an immediate refresh of Satellite repository changes into redhat.repo
subscription-manager refresh

# Verify registration status and enabled repositories
subscription-manager status
subscription-manager repos --list-enabled
subscription-manager repos --list

# Temporarily enable or disable a Satellite/RHSM repo on a single host
subscription-manager repos --enable=<repo-id>
subscription-manager repos --disable=<repo-id>

# Inspect repositories via dnf
dnf repolist
dnf repolist all
dnf repoinfo <repo-id>

# Enable or disable a custom (non-RHSM) .repo definition
dnf config-manager --enable <repo-id>
dnf config-manager --disable <repo-id>

# Unregister a host
subscription-manager unregister
```

## Custom Repositories

```bash
# Add a repository definition from a URL
dnf config-manager --add-repo https://example.com/custom/custom.repo

# Or create a .repo file manually
vi /etc/yum.repos.d/custom.repo

# Modify a repository option persistently
dnf config-manager --setopt=custom-apps.enabled=0 --save

# Remove a repository definition and clear cached metadata
rm /etc/yum.repos.d/<name>.repo
dnf clean all
```

### Repository File Format

```ini
[repo-id]
name=Repository Display Name
baseurl=https://satellite.example.com/pulp/content/ORG/Library/custom/path/
enabled=1
gpgcheck=1
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-custom
sslverify=1
```

| Parameter | Description |
|---|---|
| `[repo-id]` | Unique repository identifier used in `dnf` commands. |
| `name` | Human-readable repository description. |
| `baseurl` | Repository URL, often a Satellite content URL for internal repositories. |
| `enabled` | `1` enables the repository; `0` disables it. |
| `gpgcheck` | `1` verifies RPM signatures; `0` skips RPM signature validation. |
| `gpgkey` | Location of the public GPG key used to verify package signatures. |
| `sslverify` | `1` validates the repository server certificate; `0` skips TLS certificate validation. |

{: .warning }
> Do not disable `gpgcheck` or `sslverify` unless specifically authorized by organizational policy and the associated risk has been accepted. Prefer signed packages and validated TLS certificates.

In a Satellite-managed environment, **Content Views** control which repositories and package versions are available to hosts, and **Activation Keys** associate subscriptions, lifecycle environments, and content settings during host registration.

## STIG-Aligned GPG Keys and Custom Package Signing

{: .warning }
> ### FIPS and STIG Key-Type Requirement
>
> RHEL systems operating in FIPS mode and DISA STIG-aligned environments should not use Ed25519 (EdDSA) GPG keys for RPM package signing. Use RSA keys, such as RSA 4096-bit keys, for package-signing workflows.

```bash
# 1. Generate an RSA 4096-bit GPG key (select "(1) RSA and RSA", key size 4096)
gpg --full-generate-key

# 2. Export the public key
gpg --armor --export "admin@example.com" > RPM-GPG-KEY-custom

# 3. Configure RPM to use the signing key in ~/.rpmmacros
echo '%_gpg_name admin@example.com' >> ~/.rpmmacros

# 4. Sign and verify custom RPMs
rpmsign --addsign /path/to/my-package-1.0-1.x86_64.rpm
rpm -K /path/to/my-package-1.0-1.x86_64.rpm
```

### Trust the Public Key on Hosts & Standalone Repos

- **In Satellite:** Upload `RPM-GPG-KEY-custom` under **Content** → **Content Credentials** and attach it to the Product or Repository.
- **Directly on a host or standalone repo:**

```bash
# Import a GPG key directly on a host
cp RPM-GPG-KEY-custom /etc/pki/rpm-gpg/
rpm --import /etc/pki/rpm-gpg/RPM-GPG-KEY-custom

# Import the Red Hat release key, list imported keys, or verify a package
rpm --import /etc/pki/rpm-gpg/RPM-GPG-KEY-redhat-release
rpm -qa gpg-pubkey*
rpm -K package.rpm

# Create repository metadata for a standalone repo outside Satellite
createrepo_c /path/to/repo
```

## Exporting and Importing a Private GPG Key

{: .warning }
> A private GPG key can sign packages as its associated identity. Export it only when necessary for an authorized backup, recovery, or controlled key migration. Store and transfer the export only through an approved protected method.

```bash
# 1. Identify the key fingerprint and ID
gpg --list-secret-keys --keyid-format LONG
gpg --fingerprint "admin@example.com"

# 2. Export private key, public key, and owner trust with restrictive permissions
gpg --armor --output RPM-GPG-KEY-custom-private.asc --export-secret-keys "admin@example.com"
chown root:root RPM-GPG-KEY-custom-private.asc
chmod 0600 RPM-GPG-KEY-custom-private.asc

gpg --armor --output RPM-GPG-KEY-custom.asc --export "admin@example.com"
gpg --export-ownertrust > ownertrust.txt
chmod 0600 ownertrust.txt
```

On the destination server (after transferring via an approved protected mechanism):

```bash
# Import keys and owner trust
gpg --import RPM-GPG-KEY-custom-private.asc
gpg --import RPM-GPG-KEY-custom.asc
gpg --import-ownertrust ownertrust.txt

# Verify fingerprint and test signing
gpg --list-secret-keys --keyid-format LONG
gpg --fingerprint "admin@example.com"
rpmsign --addsign /path/to/my-package-1.0-1.x86_64.rpm
rpm -K /path/to/my-package-1.0-1.x86_64.rpm

# Securely remove temporary export files
rm -f RPM-GPG-KEY-custom-private.asc ownertrust.txt
```

{: .note }
> The GPG passphrase remains necessary to use a protected private key after import. Verify the key fingerprint through an independent authorized channel before using it to sign production RPMs.

## Package Management with `dnf`

```bash
# Install a single package (prompts for confirmation)
dnf install <pkg>

# Install multiple packages at once
dnf install pkg1 pkg2 pkg3

# Install without prompting for confirmation
dnf install -y <pkg>

# Remove a package and its unused dependencies
dnf remove <pkg>

# Remove orphaned dependency packages no longer needed by any installed package
dnf autoremove

# Check for available updates without installing them (exits 100 if updates exist)
dnf check-update

# Update all installed packages
dnf update

# Update a specific package
dnf update <pkg>

# Clear all cached packages and repository metadata
dnf clean all

# Clear cached repository metadata only
dnf clean metadata

# Download and rebuild the repository metadata cache
dnf makecache
```

### Search and Query Packages

```bash
# Search package names and summaries
dnf search <keyword>

# Search package names, summaries, and full descriptions
dnf search all <keyword>

# Display detailed package metadata (version, repo, size, description)
dnf info <pkg>

# Find which package provides a file or command (e.g., semanage)
dnf provides */<filename>
dnf provides /usr/sbin/semanage

# List all packages (installed + available) or a specific package
dnf list
dnf list <package>

# List only installed, available, or updatable packages
dnf list installed
dnf list available
dnf list updates

# Filter available or installed packages by pattern
dnf list available <pattern>
dnf list installed <pattern>

# Quickly search locally installed RPMs or list files installed by a package
rpm -qa | grep <pattern>
rpm -ql <pkg>

# Query repository metadata without installing (files, dependencies, or repo contents)
dnf repoquery -l <package-name>
dnf repoquery --requires <package-name>
dnf repoquery --repo=<repo-name>
```

## Package Groups

Package groups are collections of related packages for a common function, such as **Development Tools**, **Web Server**, or **System Tools**.

```bash
dnf group list
dnf group list hidden
dnf group info "Group Name"
dnf group install "Group Name"
```

## Module Streams

Application Streams allow selection among major software versions, such as PHP or Node.js.

| Term | Meaning |
|---|---|
| Module | An application set, such as `php` or `nodejs`. |
| Stream | A major version within a module, such as `php:8.2`. |
| Profile | A package collection for a use case, such as `common`, `minimal`, or `devel`. |

{: .note }
> Only one stream can be active for a module at a time.

```bash
# General module commands
dnf module list
dnf module list <module-name>
dnf module enable <module>:<stream>
dnf module install <module>:<stream>/<profile>
dnf module reset <module-name>

# Switch an existing PHP installation to stream 8.2
# ([d] = default, [e] = enabled, [i] = installed)
dnf module list php
dnf module reset php
dnf module enable php:8.2
dnf distro-sync

# Or install a stream directly if not yet installed
dnf install @php:8.2
```

## RPM Commands

Use `dnf` for dependency-aware installation and updates; use `rpm` primarily for local RPM inspection and integrity verification:

```bash
# Query an uninstalled RPM file
rpm -qp package.rpm

# List files installed by a package
rpm -ql package-name

# Verify an installed package or all installed packages
rpm -V package-name
rpm -Va

# Verify an RPM file signature
rpm -K package.rpm
```

## Transaction History

```bash
dnf history
dnf history info <id>
dnf history undo <id>
dnf history redo <id>
```

## Satellite Server Package Management and Updates

{: .important }
> Satellite servers lock base OS packages (via `foreman-protector`) to protect Satellite dependencies. Do not run direct `dnf update` or `dnf install` on a Satellite server—use `satellite-maintain` instead.

```bash
# Run Satellite health checks and update Satellite packages
satellite-maintain health check
satellite-maintain packages update

# Install third-party or custom packages (e.g., an antivirus agent from a custom repo) on the Satellite server
satellite-maintain packages install <package-name>

# Only if manual dnf/yum intervention is required: unlock, run dnf, then immediately re-lock
satellite-maintain packages unlock
satellite-maintain packages lock
```