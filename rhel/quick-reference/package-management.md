---
layout: default
title: "Package & Satellite Management"
parent: Quick Reference
grand_parent: RHEL
---

# Package and Satellite Management

Daily package management with `dnf` and `rpm`, module streams, custom repositories, STIG-aligned GPG signing, and Red Hat Satellite 6.19 maintenance.

> [!NOTE]
> On RHEL 8 and RHEL 9, `yum` is a symbolic link to `dnf`; therefore, `yum` and `dnf` package-management commands use the same underlying tooling. This guide uses `dnf` as the primary command.

## Satellite Server Registration and Required Repositories

If a Satellite 6.19 server running on RHEL 9 must be rebuilt, register it to Red Hat CDN and enable only the required repositories.

```bash
# 1. Register the host to Red Hat
subscription-manager register --username <rh_user> --auto-attach

# 2. Disable all repositories first to avoid conflicts
subscription-manager repos --disable "*"

# 3. Enable required RHEL 9 and Satellite 6.19 repositories
subscription-manager repos \
  --enable=rhel-9-for-x86_64-baseos-rpms \
  --enable=rhel-9-for-x86_64-appstream-rpms \
  --enable=satellite-6.19-for-rhel-9-x86_64-rpms \
  --enable=satellite-maintenance-6.19-for-rhel-9-x86_64-rpms
```

### Required Repositories for Satellite 6.19 Maintenance

If `satellite-maintain packages update` fails due to missing repositories or dependencies, verify the following repositories are enabled:

```text
rhel-9-for-x86_64-baseos-rpms
rhel-9-for-x86_64-appstream-rpms
satellite-6.19-for-rhel-9-x86_64-rpms
satellite-maintenance-6.19-for-rhel-9-x86_64-rpms
```

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
hammer subscription upload --file ~/manifest.zip --organization "Your_Org"
```

### Refresh a Manifest

Refresh the manifest after adding or renewing subscriptions:

```bash
hammer subscription refresh-manifest --organization "Your_Org"
```

## Repository Management

### View and Manage Repositories on Hosts

| Task | Command |
|---|---|
| List enabled repositories | `dnf repolist` |
| List enabled and disabled repositories | `dnf repolist all` |
| List enabled Red Hat or Satellite repositories | `subscription-manager repos --list-enabled` |
| List all subscription-managed repositories | `subscription-manager repos --list` |
| Enable a repository through `subscription-manager` | `subscription-manager repos --enable=<repo-id>` |
| Disable a repository through `subscription-manager` | `subscription-manager repos --disable=<repo-id>` |
| Enable a `.repo` repository definition | `dnf config-manager --enable <repo-id>` |
| Disable a `.repo` repository definition | `dnf config-manager --disable <repo-id>` |
| Verify repository accessibility | `dnf repolist` |
| Show repository information | `dnf repoinfo <repo-id>` |

### Register and Manage Subscriptions

| Task | Command |
|---|---|
| Register system with Red Hat | `subscription-manager register` |
| Unregister system | `subscription-manager unregister` |
| List available subscriptions | `subscription-manager list --available` |
| Automatically attach a subscription | `subscription-manager attach --auto` |
| Attach a specific subscription pool | `subscription-manager attach --pool=POOL_ID` |
| Show consumed subscriptions | `subscription-manager list --consumed` |
| Show subscription status | `subscription-manager status` |

## Custom Repositories

### Add a Repository

Add a repository definition from a URL:

```bash
dnf config-manager --add-repo https://example.com/custom/custom.repo
```

Alternatively, create a repository configuration file manually:

```bash
vi /etc/yum.repos.d/custom.repo
```

Example `/etc/yum.repos.d/custom.repo`:

```ini
[custom-apps]
name=Custom Internal Apps
baseurl=https://repo.example.com/custom-apps/el9/x86_64/
enabled=1
gpgcheck=1
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-custom
```

### Modify a Repository

Edit the appropriate file under `/etc/yum.repos.d/`, or update an option with `dnf config-manager`:

```bash
dnf config-manager --setopt=custom-apps.enabled=0 --save
```

### Remove a Repository

Remove an accidentally created repository definition and clear cached metadata:

```bash
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

> [!WARNING]
> Do not disable `gpgcheck` or `sslverify` unless specifically authorized by organizational policy and the associated risk has been accepted. Prefer signed packages and validated TLS certificates.

### Satellite Repository Management

In a Satellite-managed environment:

- **Content Views** control which repositories and package versions are available to hosts.
- **Activation Keys** associate subscriptions, lifecycle environments, and content settings during host registration.
- Use the Satellite Web UI or Hammer CLI to manage repository content, Content Views, lifecycle environments, and host registration configuration.

## STIG-Aligned GPG Keys and Custom Package Signing

> [!WARNING]
> ### FIPS and STIG Key-Type Requirement
>
> RHEL systems operating in FIPS mode and DISA STIG-aligned environments should not use Ed25519 (EdDSA) GPG keys for RPM package signing. Use RSA keys, such as RSA 4096-bit keys, for package-signing workflows.

### Generate an RSA GPG Key

```bash
gpg --full-generate-key

# Select:
# (1) RSA and RSA
# Key size: 4096
# Expiration: As required by organizational policy
# Name and email: As appropriate for the signing identity
```

### Export the Public Key

```bash
gpg --armor --export "admin@example.com" > RPM-GPG-KEY-custom
```

### Configure RPM to Use the Signing Key

Add the signing identity to `~/.rpmmacros`:

```text
%_gpg_name admin@example.com
```

### Sign and Verify Custom RPMs

Sign custom RPMs before uploading them to Satellite or a local repository:

```bash
rpmsign --addsign /path/to/my-package-1.0-1.el9.x86_64.rpm
rpm -K /path/to/my-package-1.0-1.el9.x86_64.rpm
```

### Trust the Public Key on Hosts

**In Satellite:**

1. Upload `RPM-GPG-KEY-custom` under **Content** → **Content Credentials**.
2. Attach the content credential to the applicable custom Product or Repository.

**Directly on a host:**

```bash
cp RPM-GPG-KEY-custom /etc/pki/rpm-gpg/
rpm --import /etc/pki/rpm-gpg/RPM-GPG-KEY-custom
```

If hosting a standalone repository outside Satellite, create repository metadata:

```bash
createrepo_c /path/to/repo
```

### GPG Key Management

| Task | Command |
|---|---|
| Import a GPG key | `rpm --import /etc/pki/rpm-gpg/RPM-GPG-KEY-redhat-release` |
| List imported public keys | `rpm -qa gpg-pubkey*` |
| Verify an RPM package signature | `rpm -K package.rpm` |

## Exporting and Importing a Private GPG Key

> [!WARNING]
> A private GPG key can sign packages as its associated identity. Export it only when necessary for an authorized backup, recovery, or controlled key migration. Store and transfer the export only through an approved protected method, and do not place it in a repository, shared directory, ticket, email, or other unprotected location. GnuPG specifically warns that exporting secret keys creates security risk if they are transmitted insecurely. 

### Identify the Key to Export

List secret keys and record the full fingerprint or key ID for the signing key:

```bash
gpg --list-secret-keys --keyid-format LONG
gpg --fingerprint "admin@example.com"
```

### Export the Private Key

Export the private key in ASCII-armored format:

```bash
gpg --armor \
  --output RPM-GPG-KEY-custom-private.asc \
  --export-secret-keys "admin@example.com"
```

The resulting file contains private key material. Protect the file with restrictive ownership and permissions while it exists:

```bash
chown root:root RPM-GPG-KEY-custom-private.asc
chmod 0600 RPM-GPG-KEY-custom-private.asc
```

Optionally export the corresponding public key separately:

```bash
gpg --armor \
  --output RPM-GPG-KEY-custom.asc \
  --export "admin@example.com"
```

Optionally export owner-trust settings when migrating the complete GnuPG trust configuration:

```bash
gpg --export-ownertrust > ownertrust.txt
chmod 0600 ownertrust.txt
```

GnuPG supports exporting secret keys with `--export-secret-keys` and importing keys with `--import`; exporting owner trust is useful when reproducing trust settings on another controlled system. 

### Import the Key on Another Authorized Server

1. Transfer the private-key export through an approved protected mechanism.
2. On the destination server, import the private key:

```bash
gpg --import RPM-GPG-KEY-custom-private.asc
```

3. Import the public key if it was transferred separately:

```bash
gpg --import RPM-GPG-KEY-custom.asc
```

4. Restore owner-trust settings, if exported:

```bash
gpg --import-ownertrust ownertrust.txt
```

5. Verify the imported private key and fingerprint:

```bash
gpg --list-secret-keys --keyid-format LONG
gpg --fingerprint "admin@example.com"
```

6. Confirm the imported key can sign an RPM:

```bash
rpmsign --addsign /path/to/my-package-1.0-1.el9.x86_64.rpm
rpm -K /path/to/my-package-1.0-1.el9.x86_64.rpm
```

### Cleanup

After confirming the destination server can sign packages and the export is retained only in an authorized backup location, securely remove temporary private-key export files according to organizational media-handling procedures:

```bash
rm -f RPM-GPG-KEY-custom-private.asc ownertrust.txt
```

> [!NOTE]
> The GPG passphrase remains necessary to use a protected private key after import. Verify the key fingerprint through an independent authorized channel before using it to sign production RPMs.

## Package Management with `dnf`

### Standard Package Operations

| Task | Command |
|---|---|
| Install a package | `dnf install <pkg>` |
| Install multiple packages | `dnf install pkg1 pkg2 pkg3` |
| Install without prompting | `dnf install -y <pkg>` |
| Remove a package | `dnf remove <pkg>` |
| Remove unneeded dependencies | `dnf autoremove` |
| Check for available updates | `dnf check-update` |
| Update all packages | `dnf update` |
| Update a specific package | `dnf update <pkg>` |
| Clear all cached data and metadata | `dnf clean all` |
| Clear metadata only | `dnf clean metadata` |
| Rebuild the package metadata cache | `dnf makecache` |

### Search for Packages

| Task | Command |
|---|---|
| Search package names and descriptions | `dnf search <keyword>` |
| Search all fields | `dnf search all <keyword>` |
| Show package details | `dnf info <pkg>` |
| Find a package that provides a file or command | `dnf provides */<filename>` |
| Find the package providing `semanage` | `dnf provides /usr/sbin/semanage` |
| List all packages | `dnf list` |
| List a specific package | `dnf list <package>` |
| List installed packages | `dnf list installed` |
| List available packages | `dnf list available` |
| List available updates | `dnf list updates` |
| List available packages matching a pattern | `dnf list available <pattern>` |
| List installed packages matching a pattern | `dnf list installed <pattern>` |
| Quickly search locally installed RPMs | `rpm -qa \| grep <pattern>` |
| List files installed by a package | `rpm -ql <pkg>` |
| List package files through repository metadata | `dnf repoquery -l <package-name>` |
| Show package dependencies | `dnf repoquery --requires <package-name>` |
| List packages in a repository | `dnf repoquery --repo=<repo-name>` |

Example:

```bash
dnf provides */semanage
dnf install policycoreutils-python-utils
```

## Package Groups

Package groups are collections of related packages for a common function, such as **Development Tools**, **Web Server**, or **System Tools**.

| Task | Command |
|---|---|
| List package groups | `dnf group list` |
| List all groups, including hidden groups | `dnf group list hidden` |
| Show package-group information | `dnf group info "Group Name"` |
| Install a package group | `dnf group install "Group Name"` |

## Module Streams

Application Streams allow selection among major software versions, such as PHP or Node.js.

### Module Terms

| Term | Meaning |
|---|---|
| Module | An application set, such as `php` or `nodejs`. |
| Stream | A major version within a module, such as `php:8.2`. |
| Profile | A package collection for a use case, such as `common`, `minimal`, or `devel`. |

> [!NOTE]
> Only one stream can be active for a module at a time.

### Manage a PHP Module Stream

```bash
# 1. List available streams
# [d] = default, [e] = enabled, [i] = installed
dnf module list php

# 2. Reset the currently enabled stream
dnf module reset php

# 3. Enable the desired stream
dnf module enable php:8.2

# 4. Synchronize installed packages to the selected stream
dnf distro-sync
```

If PHP is not installed, install the selected module stream after enabling it:

```bash
dnf install @php:8.2
```

### General Module Commands

| Task | Command |
|---|---|
| List modules | `dnf module list` |
| Show available streams for a module | `dnf module list <module-name>` |
| Enable a module stream | `dnf module enable <module>:<stream>` |
| Install a module profile | `dnf module install <module>:<stream>/<profile>` |
| Reset a module stream selection | `dnf module reset <module-name>` |

## RPM Commands

Use `dnf` for most package-management tasks because it resolves dependencies and works with configured repositories. Use `rpm` primarily for local RPM inspection, querying package contents, and integrity verification.

| Task | Command |
|---|---|
| Query an uninstalled RPM file | `rpm -qp package.rpm` |
| List files installed by a package | `rpm -ql package-name` |
| Verify an installed package | `rpm -V package-name` |
| Verify all installed packages | `rpm -Va` |
| Verify an RPM file signature | `rpm -K package.rpm` |

## Transaction History

`dnf history` and `yum history` are equivalent on RHEL 8 and RHEL 9.

| Task | Command |
|---|---|
| List past transactions | `dnf history` |
| Show details for a transaction | `dnf history info <id>` |
| Undo a transaction | `dnf history undo <id>` |
| Redo a transaction | `dnf history redo <id>` |

## Satellite Server Updates

> [!IMPORTANT]
> Do not use direct `dnf update` for Satellite server updates when Satellite documentation and maintenance tooling require `satellite-maintain`. Direct updates can affect Satellite dependencies.

Use `satellite-maintain` for Satellite server health checks and package updates:

```bash
satellite-maintain health check
satellite-maintain packages update
```

## Operational Tips

- Use `dnf` or `yum` for normal package installation, removal, updates, and dependency resolution.
- Use `rpm` for querying local RPM files and validating package integrity.
- Use `dnf history` to inspect, undo, or replay prior package transactions.
- Use `dnf provides */<filename>` to identify the package that supplies a needed command or file.
- Use signed RPMs and `gpgcheck=1` for custom repositories whenever possible.
- Before Satellite maintenance, verify repository availability, content synchronization status, and the results of `satellite-maintain health check`.
