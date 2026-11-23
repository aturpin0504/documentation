---
layout: default
title: Delegating KDS Root-Key Read Access
parent: Active Directory
nav_order: 1
---

# Delegating KDS Root-Key Read Access

## Purpose

This procedure delegates the Active Directory permissions required for an
approved security group to retrieve KDS root-key information using:

```powershell
Get-KdsRootKey
```

The procedure is intended for authorized Active Directory administrators who
need to grant a defined administrative group read access to existing and future
KDS root-key objects.

## Scope and security considerations

KDS root-key data is security-sensitive. Members of the delegated group can
retrieve confidential forest KDS root-key material.

- Use an approved, tightly controlled security group.
- Run the delegation in **report-only mode** before making changes.
- Review the planned access control entries (ACEs) through the applicable
  change-management process.
- Do not display, export, store, or transmit the `KeyValue` property.
- Use only sanitized placeholders in documentation; do not commit production
  domain names, domain controller names, group SIDs, KDS root-key GUIDs,
  replication USNs, or screenshots containing environment-specific details.
- Validate markings and handling requirements through the appropriate security,
  records, and release authorities.

## What the script does

The script performs the following actions for one specified Active Directory
group:

1. **Resolves the target group**
   - Queries the specified group domain.
   - Resolves the target group to its security identifier (SID).

2. **Locates the KDS configuration**
   - Queries a writable domain controller in the forest-root domain.
   - Builds the distinguished name (DN) of the KDS root-key container:

   ```text
   CN=Master Root Keys,
   CN=Group Key Distribution Service,
   CN=Services,
   CN=Configuration,...
   ```

3. **Discovers the current KDS root-key objects**
   - Finds all objects of class `msKds-ProvRootKey` below the KDS root-key
     container.
   - Does not require an administrator to enter a specific KDS root-key GUID.

4. **Checks the current discretionary access control lists (DACLs)**
   - Reads the DACL on the KDS root-key container.
   - Reads the DACL on each existing KDS root-key object.
   - Avoids adding an equivalent explicit Allow ACE when one already exists.

5. **Ensures required container permissions**
   - Grants `ListChildren` on the KDS root-key container so the group can
     enumerate KDS root-key objects.
   - Grants `ReadProperty` to descendant `msKds-ProvRootKey` objects.
   - Grants object-specific `ExtendedRight` / Control Access for confidential
     KDS attributes on future inheriting root-key objects.

6. **Ensures required permissions on existing root keys**
   - Grants direct `ReadProperty` on existing KDS root-key objects.
   - Grants direct object-specific `ExtendedRight` / Control Access for the
     confidential attributes required by `Get-KdsRootKey`.
   - Direct permissions are required when an existing root-key object does not
     inherit permissions from the parent KDS root-key container.

7. **Reports or applies the change**
   - With `$ApplyChanges = $false`, the script displays a change plan but does
     not modify Active Directory.
   - With `$ApplyChanges = $true`, the script commits only the missing ACEs.

The script delegates the following confidential KDS attributes:

- `msKds-RootKeyData`
- `msKds-CreateTime`
- `msKds-UseStartTime`
- `msKds-Version`
- `msKds-DomainID`

## Prerequisites

- Active Directory PowerShell module installed.
- Connectivity to a writable domain controller in the forest-root domain.
- Sufficient delegated authority to modify the DACL on:
  - the KDS root-key container; and
  - each existing `msKds-ProvRootKey` object.
- An approved target group and change authorization.
- A current backup or documented rollback process for Active Directory ACL
  changes, in accordance with local procedures.

## Configuration

Update the following values before execution:

```powershell
# FQDN of a writable domain controller in the forest-root domain.
$ForestRootDC = "<FOREST-ROOT-DC-FQDN>"

# DNS domain containing the group being delegated.
$GroupDomain = "<GROUP-DOMAIN-FQDN>"

# Target group sAMAccountName, distinguished name, GUID, or SID.
$GroupName = "<TARGET-GROUP-NAME>"

# $false = report-only; $true = commit ACL changes.
$ApplyChanges = $false
```

## Execution procedure

1. Copy the script into PowerShell ISE or another approved PowerShell host.
2. Update `$ForestRootDC`, `$GroupDomain`, and `$GroupName`.
3. Confirm the initial setting is:

   ```powershell
   $ApplyChanges = $false
   ```

4. Run the complete script.
5. Review the **Planned changes** output. Confirm that:
   - the target group is correct;
   - the KDS container DN is correct;
   - direct ACEs apply only to existing KDS root-key objects;
   - inherited ACEs apply only to descendant `msKds-ProvRootKey` objects.
6. If the planned changes are approved, change the setting to:

   ```powershell
   $ApplyChanges = $true
   ```

7. Rerun the complete script.
8. Confirm the script reports that ACL changes were committed.
9. Allow or verify replication of the `nTSecurityDescriptor` attribute to the
   domain controller where access will be tested.

## Delegation script

```powershell
<#
.SYNOPSIS
Delegates read access required for Get-KdsRootKey to one Active Directory group.

.DESCRIPTION
Designed for direct copy/paste use in PowerShell ISE.

The script grants the permissions required to retrieve KDS root-key
information through Get-KdsRootKey:

1. ListChildren on the KDS root-key container.
2. ReadProperty on existing and future KDS root-key objects.
3. Object-specific ExtendedRight / Control Access for confidential
   KDS attributes used by Get-KdsRootKey:
   - msKds-RootKeyData
   - msKds-CreateTime
   - msKds-UseStartTime
   - msKds-Version
   - msKds-DomainID

The script:
- Discovers KDS root-key objects automatically.
- Addresses existing root keys that do not inherit container ACEs.
- Adds inherited ACEs for future eligible root-key objects.
- Avoids duplicate equivalent ACEs.
- Runs in report-only mode unless $ApplyChanges is set to $true.

SECURITY NOTE
This delegation permits members of the target group to retrieve
confidential forest KDS root-key material. Do not display, export,
or save the KeyValue returned by Get-KdsRootKey.
#>

# ==============================================================
# Configuration
#
# Set these values for the target environment.
# ==============================================================

# FQDN of a writable domain controller in the forest-root domain.
$ForestRootDC = "<FOREST-ROOT-DC-FQDN>"

# DNS domain containing the group to delegate.
$GroupDomain = "<GROUP-DOMAIN-FQDN>"

# Target group sAMAccountName, distinguished name, GUID, or SID.
$GroupName = "<TARGET-GROUP-NAME>"

# $false = report-only; $true = commit ACL changes.
$ApplyChanges = $false

# ==============================================================
# Initialization
# ==============================================================

Import-Module ActiveDirectory

$rootDse = Get-ADRootDSE -Server $ForestRootDC

$kdsContainerDn = (
    "CN=Master Root Keys," +
    "CN=Group Key Distribution Service," +
    "CN=Services," +
    $rootDse.ConfigurationNamingContext
)

$targetGroup = Get-ADGroup `
    -Server $GroupDomain `
    -Identity $GroupName `
    -Properties SID

$targetSid = [System.Security.Principal.SecurityIdentifier]$targetGroup.SID

Write-Host ""
Write-Host "Target group : $($targetGroup.Name)"
Write-Host "Target SID   : $($targetSid.Value)"
Write-Host "Forest DC    : $ForestRootDC"
Write-Host ""

# ==============================================================
# Schema lookups
# ==============================================================

$kdsRootKeyClass = Get-ADObject `
    -Server $ForestRootDC `
    -SearchBase $rootDse.SchemaNamingContext `
    -LDAPFilter "(lDAPDisplayName=msKds-ProvRootKey)" `
    -Properties schemaIDGUID

if ($null -eq $kdsRootKeyClass.schemaIDGUID) {
    throw "Could not retrieve the schemaIDGUID for msKds-ProvRootKey."
}

$kdsRootKeyClassGuid = [Guid]$kdsRootKeyClass.schemaIDGUID

# Attributes required for a complete Get-KdsRootKey result.
$requiredKdsAttributes = @(
    "msKds-RootKeyData",
    "msKds-CreateTime",
    "msKds-UseStartTime",
    "msKds-Version",
    "msKds-DomainID"
)

$confidentialKdsAttributes = foreach ($attributeName in $requiredKdsAttributes) {
    $attribute = Get-ADObject `
        -Server $ForestRootDC `
        -SearchBase $rootDse.SchemaNamingContext `
        -LDAPFilter "(lDAPDisplayName=$attributeName)" `
        -Properties lDAPDisplayName, schemaIDGUID, searchFlags

    if ($null -eq $attribute) {
        throw "Required KDS schema attribute was not found: $attributeName"
    }

    $searchFlags = [int]$attribute.searchFlags

    # Confidential attribute flag = 0x80.
    if (($searchFlags -band 0x80) -eq 0) {
        throw "Attribute is not confidential as expected: $attributeName"
    }

    [PSCustomObject]@{
        Name        = $attribute.lDAPDisplayName
        SchemaGuid  = [Guid]$attribute.schemaIDGUID
        SearchFlags = $searchFlags
    }
}

Write-Host "KDS root-key class GUID: $kdsRootKeyClassGuid"
Write-Host ""
Write-Host "Confidential KDS attributes requiring Control Access:"

$confidentialKdsAttributes |
    Sort-Object Name |
    Format-Table Name, SchemaGuid, SearchFlags -AutoSize

# ==============================================================
# Helper functions
# ==============================================================

function Get-DirectoryEntryDacl {
    param(
        [Parameter(Mandatory)]
        [string] $Server,

        [Parameter(Mandatory)]
        [string] $DistinguishedName
    )

    $entry = [ADSI]"LDAP://$Server/$DistinguishedName"

    $entryOptions = [System.DirectoryServices.DirectoryEntryConfiguration](
        $entry.psbase.get_Options()
    )

    $entryOptions.SecurityMasks =
        [System.DirectoryServices.SecurityMasks]::Dacl

    $entry.psbase.RefreshCache(@("nTSecurityDescriptor"))

    $dacl = $entry.psbase.get_ObjectSecurity()

    if ($null -eq $dacl) {
        throw "Could not retrieve DACL for: $DistinguishedName"
    }

    [PSCustomObject]@{
        Entry = $entry
        Dacl  = $dacl
    }
}

function Save-DirectoryEntryDacl {
    param(
        [Parameter(Mandatory)]
        $Entry,

        [Parameter(Mandatory)]
        $Dacl
    )

    $Entry.psbase.set_ObjectSecurity($Dacl)
    $Entry.psbase.CommitChanges()
}

function Test-ExplicitAce {
    param(
        [Parameter(Mandatory)]
        $Dacl,

        [Parameter(Mandatory)]
        [System.Security.Principal.SecurityIdentifier] $SID,

        [Parameter(Mandatory)]
        [System.DirectoryServices.ActiveDirectoryRights] $RequiredRight,

        [Parameter(Mandatory)]
        [System.DirectoryServices.ActiveDirectorySecurityInheritance] $RequiredInheritanceType,

        # Guid::Empty requires a broad all-properties ACE.
        [Guid] $RequiredObjectType = [Guid]::Empty,

        # Used for ACEs inherited to descendant object classes.
        [Guid] $RequiredInheritedObjectType = [Guid]::Empty
    )

    $rules = $Dacl.GetAccessRules(
        $true,   # Include explicit ACEs
        $false,  # Exclude inherited ACEs
        [System.Security.Principal.SecurityIdentifier]
    )

    $requiredMask = [int]$RequiredRight

    foreach ($rule in $rules) {
        $actualMask = [int]$rule.ActiveDirectoryRights

        $hasRequiredRight = (
            ($actualMask -band $requiredMask) -eq $requiredMask
        )

        # A blank ObjectType is a broad ACE that covers a specific
        # attribute or extended right.
        $objectTypeMatches = if ($RequiredObjectType -eq [Guid]::Empty) {
            $rule.ObjectType -eq [Guid]::Empty
        }
        else {
            (
                $rule.ObjectType -eq $RequiredObjectType -or
                $rule.ObjectType -eq [Guid]::Empty
            )
        }

        # An empty inherited-object type applies broadly to descendants.
        $inheritedObjectTypeMatches = if (
            $RequiredInheritedObjectType -eq [Guid]::Empty
        ) {
            $rule.InheritedObjectType -eq [Guid]::Empty
        }
        else {
            (
                $rule.InheritedObjectType -eq $RequiredInheritedObjectType -or
                $rule.InheritedObjectType -eq [Guid]::Empty
            )
        }

        # "All" inheritance is broader than "Descendents."
        $inheritanceMatches = (
            $rule.InheritanceType -eq $RequiredInheritanceType -or
            (
                $RequiredInheritanceType -eq
                    [System.DirectoryServices.ActiveDirectorySecurityInheritance]::Descendents -and
                $rule.InheritanceType -eq
                    [System.DirectoryServices.ActiveDirectorySecurityInheritance]::All
            )
        )

        if (
            $rule.IdentityReference.Value -eq $SID.Value -and
            $rule.AccessControlType -eq
                [System.Security.AccessControl.AccessControlType]::Allow -and
            $hasRequiredRight -and
            $objectTypeMatches -and
            $inheritanceMatches -and
            $inheritedObjectTypeMatches
        ) {
            return $true
        }
    }

    return $false
}

# ==============================================================
# Locate existing KDS root-key objects
# ==============================================================

$rootKeys = Get-ADObject `
    -Server $ForestRootDC `
    -SearchBase $kdsContainerDn `
    -SearchScope OneLevel `
    -LDAPFilter "(objectClass=msKds-ProvRootKey)"

if (-not $rootKeys) {
    throw "No KDS root-key objects were found under: $kdsContainerDn"
}

$changePlan = [System.Collections.Generic.List[object]]::new()

# ==============================================================
# 1. KDS container permissions
# ==============================================================

$containerContext = Get-DirectoryEntryDacl `
    -Server $ForestRootDC `
    -DistinguishedName $kdsContainerDn

try {
    $containerChanged = $false

    # List child KDS root-key objects.
    $hasListChildren = Test-ExplicitAce `
        -Dacl $containerContext.Dacl `
        -SID $targetSid `
        -RequiredRight (
            [System.DirectoryServices.ActiveDirectoryRights]::ListChildren
        ) `
        -RequiredInheritanceType (
            [System.DirectoryServices.ActiveDirectorySecurityInheritance]::None
        )

    if (-not $hasListChildren) {
        $rule = New-Object System.DirectoryServices.ActiveDirectoryAccessRule(
            $targetSid,
            [System.DirectoryServices.ActiveDirectoryRights]::ListChildren,
            [System.Security.AccessControl.AccessControlType]::Allow
        )

        $containerContext.Dacl.AddAccessRule($rule)
        $containerChanged = $true

        [void]$changePlan.Add([PSCustomObject]@{
            Target = "KDS Container"
            Action = "Add ListChildren / This object only"
            DN     = $kdsContainerDn
        })
    }

    # Read ordinary properties on future/inheriting root-key objects.
    $hasInheritedReadProperty = Test-ExplicitAce `
        -Dacl $containerContext.Dacl `
        -SID $targetSid `
        -RequiredRight (
            [System.DirectoryServices.ActiveDirectoryRights]::ReadProperty
        ) `
        -RequiredInheritanceType (
            [System.DirectoryServices.ActiveDirectorySecurityInheritance]::Descendents
        ) `
        -RequiredInheritedObjectType $kdsRootKeyClassGuid

    if (-not $hasInheritedReadProperty) {
        $rule = New-Object System.DirectoryServices.ActiveDirectoryAccessRule(
            $targetSid,
            [System.DirectoryServices.ActiveDirectoryRights]::ReadProperty,
            [System.Security.AccessControl.AccessControlType]::Allow,
            [Guid]::Empty,
            [System.DirectoryServices.ActiveDirectorySecurityInheritance]::Descendents,
            $kdsRootKeyClassGuid
        )

        $containerContext.Dacl.AddAccessRule($rule)
        $containerChanged = $true

        [void]$changePlan.Add([PSCustomObject]@{
            Target = "KDS Container"
            Action = "Add ReadProperty / Descendant msKds-ProvRootKey objects"
            DN     = $kdsContainerDn
        })
    }

    # Add Control Access for confidential attributes on future
    # inheriting KDS root-key objects.
    foreach ($attribute in $confidentialKdsAttributes) {
        $hasInheritedExtendedRight = Test-ExplicitAce `
            -Dacl $containerContext.Dacl `
            -SID $targetSid `
            -RequiredRight (
                [System.DirectoryServices.ActiveDirectoryRights]::ExtendedRight
            ) `
            -RequiredInheritanceType (
                [System.DirectoryServices.ActiveDirectorySecurityInheritance]::Descendents
            ) `
            -RequiredObjectType $attribute.SchemaGuid `
            -RequiredInheritedObjectType $kdsRootKeyClassGuid

        if (-not $hasInheritedExtendedRight) {
            $rule = New-Object System.DirectoryServices.ActiveDirectoryAccessRule(
                $targetSid,
                [System.DirectoryServices.ActiveDirectoryRights]::ExtendedRight,
                [System.Security.AccessControl.AccessControlType]::Allow,
                $attribute.SchemaGuid,
                [System.DirectoryServices.ActiveDirectorySecurityInheritance]::Descendents,
                $kdsRootKeyClassGuid
            )

            $containerContext.Dacl.AddAccessRule($rule)
            $containerChanged = $true

            [void]$changePlan.Add([PSCustomObject]@{
                Target = "KDS Container"
                Action = (
                    "Add ExtendedRight / {0} / Descendant msKds-ProvRootKey objects" -f
                    $attribute.Name
                )
                DN = $kdsContainerDn
            })
        }
    }

    if ($ApplyChanges -and $containerChanged) {
        Save-DirectoryEntryDacl `
            -Entry $containerContext.Entry `
            -Dacl $containerContext.Dacl
    }
}
finally {
    $containerContext.Entry.Dispose()
}

# ==============================================================
# 2. Direct permissions on existing KDS root-key objects
# ==============================================================

foreach ($rootKey in $rootKeys) {
    $keyContext = Get-DirectoryEntryDacl `
        -Server $ForestRootDC `
        -DistinguishedName $rootKey.DistinguishedName

    try {
        $keyChanged = $false

        # Broad property read on this existing root-key object.
        $hasDirectReadProperty = Test-ExplicitAce `
            -Dacl $keyContext.Dacl `
            -SID $targetSid `
            -RequiredRight (
                [System.DirectoryServices.ActiveDirectoryRights]::ReadProperty
            ) `
            -RequiredInheritanceType (
                [System.DirectoryServices.ActiveDirectorySecurityInheritance]::None
            )

        if (-not $hasDirectReadProperty) {
            $rule = New-Object System.DirectoryServices.ActiveDirectoryAccessRule(
                $targetSid,
                [System.DirectoryServices.ActiveDirectoryRights]::ReadProperty,
                [System.Security.AccessControl.AccessControlType]::Allow
            )

            $keyContext.Dacl.AddAccessRule($rule)
            $keyChanged = $true

            [void]$changePlan.Add([PSCustomObject]@{
                Target = "Existing KDS Root Key"
                Action = "Add direct ReadProperty / This object only"
                DN     = $rootKey.DistinguishedName
            })
        }

        # Control Access for each confidential KDS attribute.
        foreach ($attribute in $confidentialKdsAttributes) {
            $hasDirectExtendedRight = Test-ExplicitAce `
                -Dacl $keyContext.Dacl `
                -SID $targetSid `
                -RequiredRight (
                    [System.DirectoryServices.ActiveDirectoryRights]::ExtendedRight
                ) `
                -RequiredInheritanceType (
                    [System.DirectoryServices.ActiveDirectorySecurityInheritance]::None
                ) `
                -RequiredObjectType $attribute.SchemaGuid

            if (-not $hasDirectExtendedRight) {
                $rule = New-Object System.DirectoryServices.ActiveDirectoryAccessRule(
                    $targetSid,
                    [System.DirectoryServices.ActiveDirectoryRights]::ExtendedRight,
                    [System.Security.AccessControl.AccessControlType]::Allow,
                    $attribute.SchemaGuid
                )

                $keyContext.Dacl.AddAccessRule($rule)
                $keyChanged = $true

                [void]$changePlan.Add([PSCustomObject]@{
                    Target = "Existing KDS Root Key"
                    Action = (
                        "Add direct ExtendedRight / {0} / This object only" -f
                        $attribute.Name
                    )
                    DN = $rootKey.DistinguishedName
                })
            }
        }

        if ($ApplyChanges -and $keyChanged) {
            Save-DirectoryEntryDacl `
                -Entry $keyContext.Entry `
                -Dacl $keyContext.Dacl
        }
    }
    finally {
        $keyContext.Entry.Dispose()
    }
}

# ==============================================================
# Report results
# ==============================================================

Write-Host ""

if ($changePlan.Count -eq 0) {
    Write-Host "No missing equivalent ACEs were identified." -ForegroundColor Green
}
else {
    Write-Host "Planned changes for $($targetGroup.Name):" -ForegroundColor Yellow

    $changePlan |
        Sort-Object Target, Action |
        Format-Table Target, Action, DN -AutoSize

    if ($ApplyChanges) {
        Write-Host ""
        Write-Host "ACL changes were committed to $ForestRootDC." -ForegroundColor Green
    }
    else {
        Write-Warning (
            "REPORT-ONLY MODE: No ACLs were changed. " +
            "Set `$ApplyChanges = `$true and rerun after review."
        )
    }
}
```

## Replication validation

The ACL is stored in the `nTSecurityDescriptor` attribute of the affected
Active Directory object. Use `repadmin /showobjmeta` to confirm that the
security descriptor update has replicated to the local domain controller.

Run these commands from the domain controller whose replica state you want to
inspect.

### Check the existing KDS root-key object

```powershell
repadmin /showobjmeta $env:COMPUTERNAME `
    "CN=<KDS-ROOT-KEY-GUID>,CN=Master Root Keys,CN=Group Key Distribution Service,CN=Services,CN=Configuration,DC=<FOREST-ROOT-DOMAIN>,DC=<TLD>"
```

### Check the KDS root-key container

```powershell
repadmin /showobjmeta $env:COMPUTERNAME `
    "CN=Master Root Keys,CN=Group Key Distribution Service,CN=Services,CN=Configuration,DC=<FOREST-ROOT-DOMAIN>,DC=<TLD>"
```

### Interpret the replication metadata

In each command's output, locate the row for:

```text
nTSecurityDescriptor
```

After an ACL change replicates, expect one or more of the following values to
advance relative to the pre-change baseline:

- `Ver`
- `Org.Time/Date`
- `Org.USN`
- `Loc.USN`

The `Originating DSA` should identify the domain controller where the ACL
change originated, or an intermediate replication source. The individual
`msKds-*` attribute rows do not change merely because permissions changed;
the ACL change is represented by `nTSecurityDescriptor`.

> `repadmin /showobjmeta` confirms that the directory update replicated. It
> does not by itself prove that a test account has effective access. Perform
> the access validation below after replication completes.

## Access validation

Run the following from a new PowerShell session under an account that is a
member of the delegated group.

```powershell
$kdsKeys = Get-KdsRootKey

if (-not $kdsKeys) {
    Write-Error "Get-KdsRootKey returned no KDS root-key objects."
    return
}

$kdsKeys |
    Select-Object `
        KeyId,
        IsFormatValid,
        VersionNumber,
        CreationTime,
        EffectiveTime,
        DomainController,
        @{Name = "KeyValueLength"; Expression = {
            if ($null -ne $_.KeyValue) {
                $_.KeyValue.Length
            }
            else {
                0
            }
        }} |
    Format-List
```

### Expected validation indicators

A successful result should include values similar to:

```text
KeyId            : <KDS-ROOT-KEY-GUID>
IsFormatValid    : True
VersionNumber    : 1
CreationTime     : <actual timestamp>
EffectiveTime    : <actual timestamp>
DomainController : <directory object reference>
KeyValueLength   : 64
```

The key validation conditions are:

```text
IsFormatValid  : True
KeyValueLength : 64
```

Do not run commands that display, export, serialize, or save:

```powershell
$kdsKeys.KeyValue
```

## Troubleshooting

| Symptom | Likely condition | Recommended action |
|---|---|---|
| `Get-KdsRootKey` returns no objects | The KDS root-key object cannot be enumerated in the current directory context. | Confirm `ListChildren` access on the KDS root-key container and verify the root-key object exists. |
| `KeyValueLength` is `0` | The account lacks effective Control Access for `msKds-RootKeyData`, or the ACL update has not replicated to the servicing DC. | Verify the object-specific `ExtendedRight` ACE and check `nTSecurityDescriptor` replication metadata. |
| `KeyValueLength` is `64`, but timestamps or version are missing/defaulted | The account can read root-key bytes but lacks effective Control Access for one or more confidential KDS metadata attributes. | Verify the `ExtendedRight` ACEs for `msKds-CreateTime`, `msKds-UseStartTime`, `msKds-Version`, and `msKds-DomainID`. |
| Test works on one DC but not another | The Configuration-partition ACL update has not replicated to all relevant DCs. | Check the target DC's `nTSecurityDescriptor` metadata and replication health. |
| Test account was recently added to the delegated group | The current logon token may not include the new group membership. | Start a new logon session before retesting. |
| A permission change is planned for an unintended group | Incorrect `$GroupDomain` or `$GroupName` value. | Stop, retain report-only mode, correct the configuration, and rerun the plan. |

## Rollback

To remove delegated access, remove only the ACEs added for the approved target
group from:

1. `CN=Master Root Keys` for future inheriting KDS root-key objects; and
2. each existing `msKds-ProvRootKey` object for immediate access.

Before removing permissions, identify and record the target group SID and the
specific ACEs being removed. Validate the post-change DACL and replication
state using the same methods in this procedure.

## Sanitization requirements

Do not publish or commit:

- KDS root-key values or `KeyValue` output;
- production KDS root-key GUIDs;
- production forest, domain, NetBIOS, or domain controller names;
- group SIDs or account names;
- replication USNs, DSA GUIDs, timestamps, or topology details;
- screenshots containing environment-specific infrastructure information.