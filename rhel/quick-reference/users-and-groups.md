---
layout: default
title: "User & Group Management"
parent: Quick Reference
grand_parent: RHEL
nav_order: 4
---

# User and Group Management

Daily user, group, password aging, account lockout, and STIG-aligned configuration essentials for RHEL.

{: .note }
> Validate password, account-lockout, and mount requirements against the currently applicable organizational baseline and STIG before implementing changes in a production environment.

## Managing Users

### Create Users

```bash
# Create a user with a home directory
useradd -m jsmith

# Create a user and add them to wheel for sudo access
useradd -m -G wheel jsmith

# Create a non-interactive service account with a nologin shell
useradd -r -s /sbin/nologin svc_app

# Create a user with multiple secondary groups
useradd -m -G sales,ops linda
```

### Modify Users

```bash
# Add an existing user to wheel
# -a appends the group; omitting it replaces existing secondary groups
usermod -aG wheel jsmith

# Set secondary groups; replaces all current secondary groups
usermod -G sales,ops linda

# Set the login shell to nologin
usermod -s /sbin/nologin jsmith

# Lock or unlock a local user account
usermod -L jsmith
usermod -U jsmith
```

### Delete Users

```bash
# Delete a user but retain the home directory and mail spool
userdel jsmith

# Delete a user and remove the home directory and mail spool
userdel -r jsmith

# Force-delete a user account
userdel -f jsmith
```

### Common `useradd` Options

| Option | Description |
|---|---|
| `-m` | Create the user's home directory. |
| `-u UID` | Specify the user ID. |
| `-G groups` | Assign secondary groups. |
| `-g GID` | Specify the primary group. |
| `-c comment` | Set the GECOS comment field. |
| `-s shell` | Specify the login shell. |
| `-d dir` | Specify the home directory. |
| `-r` | Create a system account. |

{: .important }
> `usermod -aG` **appends** groups. Using `usermod -G` without `-a` replaces the user's existing secondary-group memberships.

### Restrict User Login

```bash
# Set one user's shell to nologin
usermod -s /sbin/nologin username

# Prevent non-root users from logging in
touch /etc/nologin.txt
```

{: .note }
> Users attempting to log in while `/etc/nologin.txt` exists will see the file's contents.

## Password and Account Management

### Common Password Commands

| Task | Command |
|---|---|
| Change your password | `passwd` |
| Set or change a user's password | `passwd username` |
| Check password status and aging | `passwd -S username` |
| Lock an account | `passwd -l username` |
| Unlock an account | `passwd -u username` |
| Display password and account-expiration details | `chage -l username` |
| Set password aging values | `passwd -n 30 -w 3 -x 90 linda` |
| Set account-expiration date | `chage -E 2025-12-31 username` |
| Set maximum password age | `chage -M 90 username` |
| Set minimum password age | `chage -m 7 username` |
| Set expiration-warning period | `chage -W 14 username` |
| Set inactive days after expiration | `chage -I 30 username` |
| Set password to never expire | `chage -M -1 username` |
| Set account to never expire | `chage -E -1 username` |

### Smart Card Accounts and Password Expiration

For smart-card or key-based accounts where password expiration must not affect login:

```bash
# Set password to never expire
chage -M -1 jsmith

# Ensure the account itself does not expire
chage -E -1 jsmith
```

## Allowing SSH Login When a Password Is Expired

Even for smart-card or SSH-key authentication, `sshd` runs the PAM account phase. If `pam_unix.so` detects an expired password in `/etc/shadow`, it can return `PAM_AUTHTOK_EXPIRED` and deny the SSH session.

To permit key- or smart-card-based SSH logins despite password expiration, update the `pam_unix.so` account rule in:

- `/etc/pam.d/password-auth`
- `/etc/pam.d/system-auth`

For RHEL 8 and RHEL 9 systems using `authselect`, make the change in a **custom authselect profile** rather than editing PAM files directly.

Current account rule:

```text
account     required      pam_unix.so
```

Updated account rule:

```text
account     required      pam_unix.so no_pass_expiry
```

Apply custom `authselect` profile changes:

```bash
authselect apply-changes
```

## Password Complexity and Aging Requirements

### Password Complexity: `pwquality`

Password complexity is configured in:

- `/etc/security/pwquality.conf`
- `/etc/security/pwquality.conf.d/*.conf`

Example settings:

```ini
# /etc/security/pwquality.conf.d/stig.conf

minlen = 15          # Minimum password length
dcredit = -1         # Require at least one digit
ucredit = -1         # Require at least one uppercase letter
lcredit = -1         # Require at least one lowercase letter
ocredit = -1         # Require at least one special character
difok = 8            # Minimum characters changed from previous password
maxrepeat = 3        # Maximum consecutive identical characters
dictcheck = 1        # Reject dictionary words
```

### Password Aging: `/etc/login.defs`

Default password-aging and account-creation policies are configured in `/etc/login.defs`.

```ini
PASS_MAX_DAYS   60   # Maximum days before password expiration
PASS_MIN_DAYS   1    # Minimum days between password changes
PASS_WARN_AGE   7    # Warning days before expiration
```

### Password History

On RHEL 9, password-history settings are commonly configured in:

```text
/etc/security/pwhistory.conf
```

Example:

```ini
remember = 5
```

### `enforce_for_root`

The `enforce_for_root` setting controls whether password-complexity rules apply when `root` uses `passwd`.

- When enabled, `root` must meet configured password-complexity requirements.
- When commented out or disabled, `root` can set user passwords without meeting those requirements.

Example:

```ini
#enforce_for_root
```

## Modifying `useradd` Defaults

### View and Change Defaults

```bash
# View current defaults
useradd -D

# Change default shell for newly created users
useradd -D -s /sbin/nologin

# Change default home-directory base
useradd -D -b /home
```

Other examples:

```bash
useradd -D -s /bin/zsh
useradd -D -b /users
```

### Default Configuration Files

| File or Directory | Purpose |
|---|---|
| `/etc/default/useradd` | Default values used by `useradd`, including `SHELL`, `HOME`, `INACTIVE`, `SKEL`, and `CREATE_MAIL_SPOOL`. |
| `/etc/skel/` | Template files, such as `.bashrc` and `.bash_profile`, copied into newly created user home directories. |
| `/etc/login.defs` | System-wide defaults for user creation, UID/GID ranges, UMASK, and password aging. |

## Group Management

### Common Group Commands

```bash
# Create a group
groupadd appadmins

# Create a group with a specific GID
groupadd -g 2050 appadmins

# Rename a group
groupmod -n newname oldname

# Delete a group
groupdel appadmins
```

### Add or Remove Group Members

```bash
# Add a user to a group
gpasswd -a jsmith appadmins

# Remove a user from a group
gpasswd -d jsmith appadmins

# View a group and its members; works with local and AD/SSSD groups
getent group wheel

# List group members
groupmems -g sales -l

# Add or remove a user through groupmems
groupmems -g sales -a linda
groupmems -g sales -d linda
```

{: .note }
> `/etc/group` lists explicit secondary-group members. Use `id`, `groups`, `getent group`, or `groupmems` as appropriate when investigating effective group membership.

## Account Locking with `faillock`

By default, `pam_faillock` stores failed-login tallies in `/var/run/faillock`, which is typically a temporary file system cleared at reboot. A persistent location can be configured under `/var/log/faillock`.

### Example `/etc/security/faillock.conf`

```ini
dir = /var/log/faillock
deny = 3
fail_interval = 900
unlock_time = 0
```

When using a non-default tally directory, specify it with `--dir`:

```bash
# View failed-login attempts for a user
faillock --dir /var/log/faillock --user jsmith

# Reset failed-login attempts and unlock a user
faillock --dir /var/log/faillock --user jsmith --reset

# View failure records for all users
faillock --dir /var/log/faillock
```

### Troubleshooting a Locked Account

1. Check PAM configuration for `pam_faillock.so` settings:

   ```bash
   grep "pam_faillock.so" /etc/pam.d/*
   ```

2. Check `/etc/security/faillock.conf` for a configured `dir=` setting.
3. If no custom location is configured, the default directory is generally:

   ```text
   /var/run/faillock
   ```

4. Prefer resetting the tally with `faillock --reset`:

   ```bash
   faillock --dir /var/log/faillock --user username --reset
   ```

5. Reset the user's password only if the password is unknown or otherwise requires changing:

   ```bash
   passwd username
   ```

{: .note }
> Systems configured with the `silent` option may not display lockout messages to users.

## Password and Group File Integrity

### Verify Account Database Files

```bash
# Check passwd-file integrity without making changes
pwck -r

# Check group-file integrity without making changes
grpck -r

# Compare usernames in passwd and shadow
diff \
  <(awk -F: '{print $1}' /etc/passwd | sort) \
  <(awk -F: '{print $1}' /etc/shadow | sort)

# Identify malformed passwd entries; valid records have seven fields
awk -F: 'NF != 7 {print "Line " NR ": " $0}' /etc/passwd

# Identify malformed shadow entries; valid records have nine fields
awk -F: 'NF != 9 {print "Line " NR ": " $0}' /etc/shadow
```

### Common Issues

- Each `/etc/passwd` entry requires **seven fields** separated by **six colons**.
- Each `/etc/shadow` entry requires **nine fields** separated by **eight colons**.
- A user appearing in `/etc/passwd` but not `/etc/shadow`, or vice versa, indicates out-of-sync account data.
- Use `pwck` without `-r` only when prepared to correct issues interactively.
- Use `vipw` to safely edit `/etc/passwd`.
- Use `vipw -s` to safely edit `/etc/shadow`.

Back up account files before manual editing:

```bash
cp /etc/passwd /etc/passwd.backup
cp /etc/shadow /etc/shadow.backup
cp /etc/group /etc/group.backup
cp /etc/gshadow /etc/gshadow.backup
```

## Key User and Group Configuration Files

| File or Directory | Purpose |
|---|---|
| `/etc/passwd` | Local user accounts: username, UID, primary GID, GECOS field, home directory, and login shell. |
| `/etc/shadow` | Local password hashes and password/account-aging fields. |
| `/etc/group` | Local group definitions and explicit secondary-group members. |
| `/etc/gshadow` | Secure group passwords and group-administrator information. |
| `/etc/login.defs` | System-wide defaults for user creation, UID/GID ranges, UMASK, and password aging. |
| `/etc/default/useradd` | Default `useradd` settings, including shell, home base, inactive period, and skeleton directory. |
| `/etc/skel/` | Files copied into newly created user home directories. |
| `/etc/security/pwquality.conf` | Password-complexity requirements. |
| `/etc/security/pwquality.conf.d/*.conf` | Drop-in password-complexity configuration files. |
| `/etc/security/pwhistory.conf` | Password-history settings on RHEL 9. |
| `/etc/security/faillock.conf` | Account-lockout thresholds and failed-login tally location. |
| `/etc/nologin` | Prevents non-root logins when present. |

## Practical Tips

- System accounts commonly use UIDs below `1000`; normal user accounts commonly use UIDs of `1000` or higher.
- Use `usermod -aG` to add groups without removing existing secondary-group memberships.
- Use `/sbin/nologin` for service, disabled, or non-interactive accounts when appropriate.
- Use `chage -l username` to review password and account expiration settings.
- Use `faillock --reset` instead of manually deleting tally files when possible.
- Test a user's login environment and permissions after changes:

```bash
su - username
```