---
layout: default
title: "Package Management"
parent: Quick Reference
grand_parent: RHEL
nav_order: 7
---

# Package Management

Daily package management with `dnf` and `rpm`, module streams, repository configuration, Satellite-managed host registration, and custom RPM package signing.

{: .note}
> This guide uses `dnf` as the primary package-management command. On systems where `yum` is present, it is typically a compatibility link to `dnf`.

## Satellite-Managed Host Registration and Repositories

This section applies to RHEL hosts managed by Red Hat Satellite. For Satellite server registration, repository prerequisites, manifests, maintenance, backups, updates, and upgrades, see the **Satellite Management** reference.

On Satellite-managed hosts, `subscription-manager` manages Red Hat and Satellite-provided repository entitlement data. Satellite commonly controls available content through Content Views, Lifecycle Environments, and Activation Keys.

```bash
# Register a host to Satellite using an organization and activation key
subscription-manager register \
  --org="<organization>" \
  --activationkey="<activation_key>"

# Refresh entitlement and repository data
subscription-manager refresh

# Verify registration status
subscription-manager status

# List enabled repositories
subscription-manager repos --list-enabled

# List all repositories available to the host
subscription-manager repos --list

# Temporarily enable or disable an entitled repository on the host
subscription-manager repos --enable=<repo-id>
subscription-manager repos --disable=<repo-id>

# Inspect repositories through dnf
dnf repolist
dnf repolist --all
dnf repoinfo <repo-id>

# Unregister a host
subscription-manager unregister
```

{: .note}
> `subscription-manager` regenerates `/etc/yum.repos.d/redhat.repo`. Do not manually edit that file. Manage Red Hat or Satellite-provided repositories with `subscription-manager`.

### Custom Repository Files

Use `dnf config-manager` only for approved custom, third-party, or locally managed `.repo` files.

```bash
# Add a repository definition from a URL
dnf config-manager --add-repo https://example.com/custom/custom.repo

# Create or edit a custom repository file
vi /etc/yum.repos.d/custom.repo

# Enable or disable a custom repository
dnf config-manager --enable <repo-id>
dnf config-manager --disable <repo-id>

# Modify a repository option persistently
dnf config-manager --setopt=custom-apps.enabled=0 --save

# Remove a custom repository definition and clear cached metadata
rm /etc/yum.repos.d/<name>.repo
dnf clean all
```

{: .note}
> For Satellite-managed Red Hat repositories, do not manually create or edit `redhat.repo`. Use Satellite content management and `subscription-manager`. Use local `.repo` files only for approved custom or third-party repositories.

### Repository File Format

```ini
[repo-id]
name=Repository Display Name
baseurl=https://satellite.example.example/pulp/content/<organization>/Library/custom/path/
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

{: .warning}
> Do not disable `gpgcheck` or `sslverify` unless specifically authorized by organizational policy and the associated risk has been accepted. Prefer signed packages and validated TLS certificates.

In a Satellite-managed environment, **Content Views** control which repositories and package versions are available to hosts. **Activation Keys** associate subscriptions, lifecycle environments, and content settings during host registration.

## GPG Keys and Custom RPM Package Signing

RPM signing provides package-integrity and package-origin verification. A signing workflow uses a private key to sign RPMs and distributes the corresponding public key to Satellite and client systems.

{: .warning}
> **FIPS and STIG Key-Type Requirement**
>
> For systems operating under applicable FIPS and DISA STIG requirements, use an organizationally approved RSA signing key, such as RSA 4096-bit. Do not use Ed25519 (EdDSA) signing keys unless specifically authorized by the applicable security policy. Validate key-algorithm, key-length, protection, storage, and rotation requirements with the responsible security authority.

### Signing-Key Roles

| Component | Purpose | Handling |
|---|---|---|
| Private key | Signs RPM packages. | Restrict access; store only on an approved signing host or approved protected backup location. |
| Public key | Verifies signed RPM packages. | Upload to Satellite and distribute to clients as required. |
| Signed RPM | RPM containing the signing-key signature. | Sign before publishing to a repository. |
| Fingerprint | Unique identifier for the public/private key pair. | Verify through an independent authorized channel before production use. |

### Prerequisites

Install GnuPG and the RPM signing utility on the approved signing system:

```bash
dnf install -y gnupg2 rpm-sign
```

If interactive GPG key creation or signing reports a pinentry error, install an appropriate pinentry package for the environment:

```bash
dnf install -y pinentry
```

### Generate a GPG Signing Key

Generate a key pair interactively:

```bash
gpg --full-generate-key
```

Recommended interactive selections should follow organizational policy. A common RSA signing-key configuration is:

| Prompt | Example Value |
|---|---|
| Key type | `RSA and RSA` |
| Key size | `4096` |
| Expiration | Organization-approved expiration or rotation interval |
| Real name | `<organization> RPM Repository Signing Key` |
| Email | `<shared-signing-contact@example.invalid>` |
| Comment | `<repository-or-environment>` |
| Passphrase | Use an organization-approved protected-key process |

{: .note}
> Use a role-based or infrastructure-based identity rather than an individual's name. Ensure the signing identity, recovery process, and key custodianship remain valid when personnel change.

List available secret keys and record the selected key fingerprint:

```bash
# List secret keys using long key IDs
gpg --list-secret-keys --keyid-format LONG

# Display the full fingerprint for a selected key
gpg --fingerprint "<key-id-or-identity>"
```

### Export the Public Key

Export the public key in ASCII-armored format:

```bash
# Export the selected public key
gpg --export --armor "<key-id-or-identity>" \
  > RPM-GPG-KEY-custom

# Review the exported public key
cat RPM-GPG-KEY-custom
```

Import the public key into the local RPM database when the signing host must verify packages signed by that key:

```bash
rpm --import RPM-GPG-KEY-custom

# List imported RPM public keys
rpm -qa 'gpg-pubkey*'
```

### Configure RPM to Use the Signing Key

Configure the signing identity in the signing user's `~/.rpmmacros` file:

```text
%_gpg_name <key-id-or-identity>
```

For example:

```bash
# Add the signing-key identity to the current user's RPM macro file
echo '%_gpg_name <key-id-or-identity>' >> ~/.rpmmacros

# Confirm the configured macro
grep '^%_gpg_name' ~/.rpmmacros
```

Use a key ID, fingerprint, full name, or email address that uniquely identifies the approved signing key.

### Sign and Verify Custom RPMs

Sign RPMs before uploading or publishing them:

```bash
# Sign one RPM
rpm --addsign /path/to/package-1.0-1.x86_64.rpm

# Sign multiple RPMs in the current directory
rpm --addsign ./*.rpm

# Verify an RPM signature
rpm --checksig /path/to/package-1.0-1.x86_64.rpm
```

If the private key is passphrase-protected, enter the passphrase when prompted.

{: .warning}
> Re-signing changes an RPM's checksum. Sign packages before publishing them to Satellite or another repository. If an already-published RPM must be re-signed, treat it as updated content and follow the applicable repository-content change process.

### Distribute the Public Key

The public key must be available to systems that validate packages signed by the corresponding private key.

For standalone hosts or repositories:

```bash
# Install the public key in the standard RPM GPG-key directory
install -m 0644 RPM-GPG-KEY-custom \
  /etc/pki/rpm-gpg/RPM-GPG-KEY-custom

# Import the key into the local RPM database
rpm --import /etc/pki/rpm-gpg/RPM-GPG-KEY-custom

# Verify imported RPM public keys
rpm -qa 'gpg-pubkey*'

# Verify an RPM signature
rpm --checksig package.rpm
```

For Satellite-managed repositories, upload the public key as a Content Credential and associate it with the applicable custom repository. See the **Satellite Management** reference for the Satellite Web UI and Hammer CLI procedures.

### Trust the Public Key on Hosts and Standalone Repositories

For hosts consuming content outside the normal Satellite registration workflow:

```bash
# Install the public key in the standard RPM GPG-key directory
install -m 0644 RPM-GPG-KEY-custom \
  /etc/pki/rpm-gpg/RPM-GPG-KEY-custom

# Import the key into the RPM database
rpm --import /etc/pki/rpm-gpg/RPM-GPG-KEY-custom

# Verify imported RPM public keys
rpm -qa 'gpg-pubkey*'

# Verify the signature on a package
rpm --checksig package.rpm
```

For a standalone RPM repository, create or refresh repository metadata after placing signed RPMs in the repository directory:

```bash
createrepo_c /path/to/repo
```

### Back Up a Private GPG Key

{: .warning}
> A private GPG key can sign packages as its associated identity. Export it only for an authorized backup, recovery, or controlled migration. Store and transfer it only through an approved protected method.

```bash
# Identify the key before export
gpg --list-secret-keys --keyid-format LONG
gpg --fingerprint "<key-id-or-identity>"

# Export the private key in ASCII-armored format
gpg --armor \
  --output RPM-GPG-KEY-custom-private.asc \
  --export-secret-keys "<key-id-or-identity>"

# Export the public key
gpg --armor \
  --output RPM-GPG-KEY-custom.asc \
  --export "<key-id-or-identity>"

# Export owner trust when required for controlled migration
gpg --export-ownertrust > ownertrust.txt

# Restrict locally created export files
chmod 0600 RPM-GPG-KEY-custom-private.asc
chmod 0600 ownertrust.txt
chmod 0644 RPM-GPG-KEY-custom.asc
```

Store the private-key export and any associated recovery material in an approved protected repository or vault. Retain the public key and the verified fingerprint with the key-recovery documentation.

### Import an Existing Private Key

Use this procedure only for authorized key recovery or controlled migration.

```bash
# Import the private key
gpg --import RPM-GPG-KEY-custom-private.asc

# Import the public key
gpg --import RPM-GPG-KEY-custom.asc

# Import owner trust when included in the approved backup
gpg --import-ownertrust ownertrust.txt

# Verify available secret keys and fingerprint
gpg --list-secret-keys --keyid-format LONG
gpg --fingerprint "<key-id-or-identity>"

# Test package signing and verification
rpm --addsign /path/to/package-1.0-1.x86_64.rpm
rpm --checksig /path/to/package-1.0-1.x86_64.rpm
```

After completing an authorized migration, remove temporary private-key export files from the working directory according to the organization's approved media-handling and sanitization process.

### Retire or Remove a GPG Key

Before removing a signing key, ensure that:

1. Replacement packages have been signed with the successor key, if applicable.
2. The successor public key has been distributed to Satellite and affected clients.
3. Repository associations and Content Views have been updated.
4. The key-retirement action is authorized and documented.

```bash
# List secret keys and identify the fingerprint
gpg --list-secret-keys --keyid-format LONG
gpg --fingerprint "<key-id-or-identity>"

# Delete the secret key first
gpg --delete-secret-keys "<fingerprint>"

# Delete the corresponding public key
gpg --delete-keys "<fingerprint>"

# Verify removal from the local GPG keyring
gpg --list-keys
gpg --list-secret-keys
```

Remove or disassociate the corresponding Content Credential in Satellite only after affected repositories and clients have transitioned to the replacement key or no longer require the retired key.

### GPG Key Management Quick Reference

| Task | Command |
|---|---|
| List secret signing keys | `gpg --list-secret-keys --keyid-format LONG` |
| List public keys | `gpg --list-keys` |
| Display a key fingerprint | `gpg --fingerprint "<key-id-or-identity>"` |
| Export a public key | `gpg --export --armor "<identity>" > RPM-GPG-KEY-custom` |
| Export a private key | `gpg --armor --export-secret-keys "<identity>" > RPM-GPG-KEY-custom-private.asc` |
| Import a GPG key | `gpg --import <key-file>` |
| Import a public key into RPM | `rpm --import RPM-GPG-KEY-custom` |
| List imported RPM public keys | `rpm -qa 'gpg-pubkey*'` |
| Sign an RPM | `rpm --addsign package.rpm` |
| Verify an RPM signature | `rpm --checksig package.rpm` |
| Remove default GPG configuration from RPM macros | Remove the `%_gpg_name` line from `~/.rpmmacros` |

## Package Management with `dnf`

```bash
# Install a single package
dnf install <package>

# Install multiple packages
dnf install package1 package2 package3

# Install without prompting for confirmation
dnf install -y <package>

# Remove a package and dependencies no longer required
dnf remove <package>

# Remove orphaned dependency packages
dnf autoremove

# Check for available updates without installing them
# Exit status 100 indicates updates are available
dnf check-update

# Update all installed packages
dnf update

# Update a specific package
dnf update <package>

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

# Search package names, summaries, and descriptions
dnf search all <keyword>

# Display detailed package metadata
dnf info <package>

# Find which package provides a file or command
dnf provides '*/<filename>'
dnf provides /usr/sbin/semanage

# List all packages or a specific package
dnf list
dnf list <package>

# List installed, available, or updatable packages
dnf list installed
dnf list available
dnf list updates

# Filter available or installed packages by pattern
dnf list available <pattern>
dnf list installed <pattern>

# Search locally installed RPMs
rpm -qa | grep <pattern>

# List files installed by a package
rpm -ql <package>

# Query repository metadata without installing
dnf repoquery -l <package-name>
dnf repoquery --requires <package-name>
dnf repoquery --repo=<repo-name>
```

## Package Groups

Package groups are collections of related packages for a common function, such as development tools, web servers, or system administration tools.

```bash
# List package groups
dnf group list

# List visible and hidden package groups
dnf group list hidden

# Display group details
dnf group info "Group Name"

# Install a package group
dnf group install "Group Name"
```

## Module Streams

Application Streams allow selection among major software versions, such as PHP or Node.js.

| Term | Meaning |
|---|---|
| Module | An application set, such as `php` or `nodejs`. |
| Stream | A major version within a module, such as `php:8.2`. |
| Profile | A package collection for a use case, such as `common`, `minimal`, or `devel`. |

{: .note}
> Only one stream can be active for a module at a time.

```bash
# General module commands
dnf module list
dnf module list <module-name>
dnf module enable <module>:<stream>
dnf module install <module>:<stream>/<profile>
dnf module reset <module-name>

# Switch an existing PHP installation to stream 8.2
dnf module list php
dnf module reset php
dnf module enable php:8.2
dnf distro-sync

# Install a stream directly if it is not already installed
dnf install @php:8.2
```

## RPM Commands

Use `dnf` for dependency-aware installation and updates. Use `rpm` primarily for local RPM inspection, package verification, and signature validation.

```bash
# Query an uninstalled RPM file
rpm -qp package.rpm

# List files installed by a package
rpm -ql package-name

# Verify an installed package or all installed packages
rpm -V package-name
rpm -Va

# Verify an RPM file signature
rpm --checksig package.rpm
```

## Transaction History

```bash
# List dnf transaction history
dnf history

# Display information about a transaction
dnf history info <id>

# Attempt to undo a transaction
dnf history undo <id>

# Repeat a transaction
dnf history redo <id>
```

{: .warning}
> Review the transaction details and dependency changes before using `dnf history undo`. Some transactions cannot be fully reversed if later package changes depend on them.