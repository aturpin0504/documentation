---
layout: default
title: "Satellite Management"
parent: Quick Reference
grand_parent: RHEL
nav_order: 8
---

# Red Hat Satellite Management

Satellite administration procedures for health checks, backups, package updates, patch updates, version upgrades, service management, and approved package installation.

{: .important}
> Use `satellite-maintain` for Satellite lifecycle operations. Do not run direct `dnf update`, `dnf upgrade`, or `dnf install` on a Satellite server unless an approved, version-specific maintenance procedure explicitly requires it.

## Before You Begin

Before updating, upgrading, restoring, or making a significant Satellite configuration change:

1. Confirm the maintenance window and organizational approval.
2. Confirm the target Satellite and RHEL versions are supported.
3. Verify required repositories for the installed Satellite version are enabled.
4. Create and validate a current backup.
5. Run the applicable `satellite-maintain` pre-check.
6. Use a persistent terminal session, such as `tmux`, for long-running operations.
7. Review the current Red Hat documentation and release notes for the installed and target versions.

```bash
# Start a persistent shell session before a long-running operation
tmux new -s satellite-maintenance

# Reconnect to an existing session
tmux attach -t satellite-maintenance
```

## Health Checks and Service Management

Run health checks before and after significant maintenance.

```bash
# Run Satellite health checks
satellite-maintain health check

# Display Satellite service status
satellite-maintain service status

# Stop Satellite services
satellite-maintain service stop

# Start Satellite services
satellite-maintain service start

# Restart Satellite services
satellite-maintain service restart
```

{: .note}
> Use service stop, start, and restart actions only as part of an approved troubleshooting, backup, recovery, or maintenance procedure.

## Rebuilding and Registering a Satellite Server

Use this procedure only when rebuilding or re-registering a Satellite server. Register the server to Red Hat CDN and enable only the BaseOS, AppStream, Satellite, and Satellite Maintenance repositories that match the applicable RHEL and Satellite versions.

```bash
# Register the host to Red Hat
subscription-manager register --username <rh_user> --auto-attach

# Disable all repositories first to avoid conflicts
subscription-manager repos --disable "*"

# Verify exact repository IDs for the installed RHEL and Satellite versions
subscription-manager repos --list | grep -E "baseos-rpms|appstream-rpms|satellite-"

# Enable required RHEL and Satellite repositories
subscription-manager repos \
  --enable=rhel-<rhel_ver>-for-x86_64-baseos-rpms \
  --enable=rhel-<rhel_ver>-for-x86_64-appstream-rpms \
  --enable=satellite-<sat_ver>-for-rhel-<rhel_ver>-x86_64-rpms \
  --enable=satellite-maintenance-<sat_ver>-for-rhel-<rhel_ver>-x86_64-rpms
```

If a `satellite-maintain` package, patch-update, or upgrade operation fails because of missing repositories or dependencies, verify that the required BaseOS, AppStream, Satellite, and Satellite Maintenance repositories are enabled for the installed Satellite and RHEL versions.

## Subscription Manifests

A Subscription Manifest (`.zip`) carries Red Hat entitlements and CDN certificates from the Red Hat Hybrid Cloud Console into Satellite, allowing Satellite to synchronize Red Hat repositories.

### Export a Manifest

1. Go to the [Red Hat Hybrid Cloud Console manifest page](https://console.redhat.com/subscriptions/manifests).
2. Locate or create the manifest allocation for the applicable Red Hat Satellite version.
3. Select the **ellipsis** (`⋮`) next to the manifest allocation.
4. Select **Export**.
5. Download the generated `.zip` file.
6. Import the manifest into Satellite.

### Import a Manifest into Satellite

**Web UI:**

```text
Content → Subscriptions → Manage Manifest → Upload
```

**Hammer CLI:**

```bash
# Import a manifest
hammer subscription upload \
  --file ~/manifest.zip \
  --organization "<organization>"

# Refresh after adding or renewing subscriptions
hammer subscription refresh-manifest \
  --organization "<organization>"
```

## Repository Prerequisites for Maintenance

Before a Satellite package update, patch update, or version upgrade, verify that the required BaseOS, AppStream, Satellite, and Satellite Maintenance repositories are enabled for the installed Satellite and RHEL versions.

```bash
# List enabled repositories
subscription-manager repos --list-enabled

# Search available repository IDs
subscription-manager repos --list | \
  grep -E "baseos-rpms|appstream-rpms|satellite-"

# Example: enable the Satellite Maintenance repository
# Replace placeholders with the installed versions.
subscription-manager repos --enable \
  satellite-maintenance-<satellite_version>-for-rhel-<rhel_version>-x86_64-rpms
```

{: .warning}
> Do not enable repository IDs copied from a different RHEL or Satellite version. Confirm the exact repository IDs available to the local Satellite server before changing repository configuration.

## Package Updates with `satellite-maintain packages`

Use `satellite-maintain packages` for approved package installation and package-update actions on a Satellite server. This avoids bypassing Satellite package-protection and maintenance workflows.

```bash
# Install an approved package
satellite-maintain packages install <package-name>

# Update one approved package
satellite-maintain packages update <package-name>

# Update multiple approved packages
satellite-maintain packages update <package-name-1> <package-name-2>

# Update all available packages
satellite-maintain packages update
```

{: .warning}
> Running `satellite-maintain packages update` with no package names updates all available packages. Treat this as a planned maintenance action: confirm enabled repositories, review the intended scope, create or confirm a current backup, and perform the work in an approved maintenance window.

### Normal Package-Update Workflow

```bash
# Confirm required repositories are enabled
subscription-manager repos --list-enabled

# Run a health check
satellite-maintain health check

# Create or confirm a current backup
satellite-maintain backup offline /path/to/backup-directory

# Apply an approved targeted package update
satellite-maintain packages update <package-name>

# Or, for an approved full package update
satellite-maintain packages update

# Validate Satellite after the update
satellite-maintain health check
satellite-maintain service status
```

{: .note}
> **Command selection:** Use `satellite-maintain packages update <package-name>` for an approved targeted package update. Use `satellite-maintain packages update` without package names only when intentionally updating all available packages. Use `satellite-maintain update check` and `satellite-maintain update run` for a planned Satellite patch-update workflow.

## Back Up Satellite

Create and validate a backup before a Satellite patch update, version upgrade, major configuration change, or other significant maintenance event.

### Offline Backup

An offline backup stops Satellite services to create a consistent backup.

```bash
# Display supported offline-backup options for the installed version
satellite-maintain backup offline --help

# Create an offline backup
satellite-maintain backup offline /path/to/backup-directory
```

### Incremental Backup

```bash
# Create an incremental backup based on an existing full backup
satellite-maintain backup offline \
  --incremental /path/to/full-backup \
  /path/to/backup-directory
```

{: .warning}
> Satellite backups can contain sensitive configuration and credential material. Store backups in an approved protected location with appropriate access controls. Verify backup completion and perform restoration testing according to the organization's recovery process.

### Restore Planning

Use the restore procedure that matches the installed Satellite version, database configuration, backup type, and recovery architecture.

Before restoring:

1. Confirm the backup is complete and accessible.
2. Confirm the recovery host meets the applicable Satellite installation requirements.
3. Confirm the Satellite and RHEL versions align with the backup and documented restore procedure.
4. Review the current version-specific Red Hat disaster-recovery guidance.
5. Perform recovery actions only under an approved incident-response or recovery process.

```bash
# Review locally available backup command options
satellite-maintain backup --help
```

## Satellite Patch Updates

Use this process for planned Satellite patch or z-stream updates within the same Satellite release, for example, `<version>.0` to `<version>.1`.

Satellite patch updates are distinct from targeted package updates:

| Task | Primary Command |
|---|---|
| Install an approved additional package | `satellite-maintain packages install <package-name>` |
| Update one or more approved packages | `satellite-maintain packages update <package-name>...` |
| Update all available packages under an approved maintenance plan | `satellite-maintain packages update` |
| Run a Satellite patch-update readiness check | `satellite-maintain update check` |
| Apply a Satellite patch update | `satellite-maintain update run` |

For a planned Satellite patch update, use the `satellite-maintain update` workflow.

### 1. Confirm Repositories and Backups

```bash
# Confirm required repositories are enabled
subscription-manager repos --list-enabled

# Run a health check
satellite-maintain health check

# Create an offline backup
satellite-maintain backup offline /path/to/backup-directory
```

### 2. Run the Update Pre-Check

```bash
satellite-maintain update check
```

Resolve reported errors before proceeding.

### 3. Run the Update

```bash
satellite-maintain update run
```

### 4. Validate the Update

```bash
# Confirm Satellite service health
satellite-maintain health check

# Confirm service status
satellite-maintain service status

# Review installed Satellite packages
rpm -qa | grep -E 'satellite|foreman|katello|pulp'
```

{: .note}
> Satellite patch updates can also update supported underlying RHEL packages. Follow the update output and reboot only when prompted or required by the approved maintenance procedure.

## Satellite Version Upgrades

A version upgrade moves Satellite to a new supported release, for example, from `<source_version>` to `<target_version>`.

{: .warning}
> A Satellite version upgrade is a planned change, not a routine patch action. Use the current Red Hat Satellite Upgrade Helper, release notes, upgrade documentation, and organization-approved change process to generate version-specific instructions.

### High-Level Upgrade Workflow

1. Identify the current and target Satellite versions.
2. Review the target-version release notes, prerequisites, and known issues.
3. Confirm supported RHEL and database requirements.
4. Create and validate an offline backup.
5. Enable only the repositories specified for the target version.
6. Run the upgrade pre-check.
7. Run the upgrade.
8. Complete post-upgrade validation, including services, content synchronization, Content Views, Capsules, templates, and integrations.

```bash
# Run a version-specific pre-upgrade check
satellite-maintain upgrade check --target-version <target_version>

# Run the version upgrade
satellite-maintain upgrade run --target-version <target_version>
```

After an upgrade:

```bash
# Run health checks
satellite-maintain health check

# Verify services
satellite-maintain service status

# Confirm enabled repositories
subscription-manager repos --list-enabled
```

{: .note}
> Use a persistent terminal session such as `tmux` during an upgrade. Do not rely on a workstation session remaining connected for the full duration of a long-running maintenance operation.

## Approved Package Installation

Install additional packages on a Satellite server only when required, compatible with the installed Satellite release, and approved through the applicable change process.

```bash
# Install an approved package through satellite-maintain
satellite-maintain packages install <package-name>
```

If direct package-manager intervention is specifically required by an approved procedure:

```bash
# Temporarily unlock protected package management
satellite-maintain packages unlock

# Perform only the approved package-management action
dnf install <package-name>

# Re-enable package protection immediately afterward
satellite-maintain packages lock
```

{: .warning}
> Do not run direct `dnf install`, `dnf update`, or `dnf upgrade` on a Satellite server for routine maintenance. Use `satellite-maintain packages` or the applicable `satellite-maintain update` workflow. Unlock package protection and use `dnf` only when an approved, version-specific procedure explicitly directs it.

### Upload the Public Key to Satellite

Create a GPG Content Credential in Satellite:

```text
Content → Content Credentials → Create Content Credential
```

Use the following values:

| Field | Value |
|---|---|
| Name | `<custom-repository-gpg-key>` |
| Type | `GPG Key` |
| Content | Paste or upload the contents of `RPM-GPG-KEY-custom`. |

Then associate the content credential with the applicable custom product or repository.

```text
Content → Products → <product> → Repositories → <repository>
```

Use the Satellite UI or approved automation to associate the GPG key with the repository and publish the updated Content View as required by the local content-management process.

**Hammer CLI example:**

```bash
hammer content-credentials create \
  --content-type gpg_key \
  --name "<custom-repository-gpg-key>" \
  --organization "<organization>" \
  --path RPM-GPG-KEY-custom
```

## Maintenance Quick Reference

| Task | Command |
|---|---|
| Run health checks | `satellite-maintain health check` |
| Check service status | `satellite-maintain service status` |
| Start services | `satellite-maintain service start` |
| Stop services | `satellite-maintain service stop` |
| Restart services | `satellite-maintain service restart` |
| View offline-backup options | `satellite-maintain backup offline --help` |
| Create an offline backup | `satellite-maintain backup offline /path/to/backup-directory` |
| Create an incremental backup | `satellite-maintain backup offline --incremental <full-backup> <destination>` |
| Install an approved package | `satellite-maintain packages install <package-name>` |
| Update an approved package | `satellite-maintain packages update <package-name>` |
| Update all available packages | `satellite-maintain packages update` |
| Run patch-update pre-check | `satellite-maintain update check` |
| Run a Satellite patch update | `satellite-maintain update run` |
| Run version-upgrade pre-check | `satellite-maintain upgrade check --target-version <version>` |
| Run a version upgrade | `satellite-maintain upgrade run --target-version <version>` |
| Unlock package protection temporarily | `satellite-maintain packages unlock` |
| Re-enable package protection | `satellite-maintain packages lock` |