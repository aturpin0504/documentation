# User & Group Management

Daily user, group, password aging, `faillock`, and STIG configuration essentials for RHEL.

---

## Managing Users (`useradd`, `usermod`, `userdel`)

```bash
# Create a user with a home directory
useradd -m jsmith

# Create a user and add them to wheel (sudo access)
useradd -m -G wheel jsmith

# Create a non-interactive / STIG-compliant service account with nologin shell
useradd -r -s /sbin/nologin svc_app

# Add an existing user to wheel (-a = append; NEVER omit -a or you overwrite other groups!)
usermod -aG wheel jsmith

# Set an existing user's shell to nologin (STIG requirement for non-interactive/disabled accounts)
usermod -s /sbin/nologin jsmith

# Lock / unlock a local user account
usermod -L jsmith
usermod -U jsmith

# Delete a user (keeps home directory and mail spool)
userdel jsmith

# Delete a user AND remove their home directory and mail spool
userdel -r jsmith
```

---

## Password Management & Smart Card Expiration

```bash
passwd jsmith            # Set or change a user's password
passwd -S jsmith         # Check password status (locked, set, aging)
chage -l jsmith          # View password and account expiration details
chage -M -1 jsmith       # Set password to NEVER expire (common for smart card accounts)
chage -E -1 jsmith       # Ensure the account itself never expires
```

### Allowing SSH Login When a Password Is Expired (PAM)

Even when using **smart cards** or **SSH keys**, `sshd` still runs the PAM `account` phase. If `pam_unix.so` sees an expired password in `/etc/shadow`, it returns `PAM_AUTHTOK_EXPIRED` and drops the SSH connection.

To allow key/smart-card SSH logins regardless of `/etc/shadow` password expiration:

1. In `/etc/pam.d/password-auth` and `/etc/pam.d/system-auth` (or in your custom `authselect` profile so updates don't overwrite it), locate the `account` line for `pam_unix.so`:
   ```text
   account     required      pam_unix.so
   ```
2. Append `no_pass_expiry`:
   ```text
   account     required      pam_unix.so no_pass_expiry
   ```

*(If you use `authselect` on RHEL 8/9, apply this inside a custom `authselect` profile rather than editing `/etc/pam.d/*` directly, then run `authselect apply-changes`.)*

---

## STIG Password Requirements

Password complexity and aging rules are controlled in two main places:

- **`/etc/security/pwquality.conf`** (or `/etc/security/pwquality.conf.d/*.conf`) — Password complexity rules enforced by `pam_pwquality`:
  ```ini
  minlen = 15          # Minimum password length
  dcredit = -1         # Require at least 1 digit
  ucredit = -1         # Require at least 1 uppercase letter
  lcredit = -1         # Require at least 1 lowercase letter
  ocredit = -1         # Require at least 1 special character
  difok = 8            # Minimum characters changed from old password
  maxrepeat = 3        # Max consecutive identical characters
  dictcheck = 1        # Reject dictionary words
  ```
- **`/etc/login.defs`** — Default aging and account creation policies:
  ```text
  PASS_MAX_DAYS   60   # Max days before password expires (STIG)
  PASS_MIN_DAYS   1    # Min days between password changes
  PASS_WARN_AGE   7    # Days before expiration to warn user
  ```
- **`/etc/security/pwhistory.conf`** (RHEL 9) — Password reuse history (`remember = 5`).

---

## Modifying `useradd` Defaults

```bash
useradd -D                     # View current useradd defaults
useradd -D -s /sbin/nologin    # Change default shell for new users
useradd -D -b /home            # Change default base home directory
```

You can also edit `/etc/default/useradd` directly (`SHELL`, `HOME`, `INACTIVE`, `SKEL`, `CREATE_MAIL_SPOOL`). Files placed in `/etc/skel/` are automatically copied into every new user's home directory.

---

## Common Group Management Tasks

```bash
groupadd appadmins             # Create a group
groupadd -g 2050 appadmins     # Create a group with a specific GID
groupmod -n newname oldname    # Rename a group
groupdel appadmins             # Delete a group
gpasswd -a jsmith appadmins    # Add a user to a group
gpasswd -d jsmith appadmins    # Remove a user from a group
getent group wheel             # View group members (works for local + AD/SSSD groups)
```

---

## `faillock` (STIG Persistent Directory)

By default, `pam_faillock` stores failure tallies in `/var/run/faillock` (`tmpfs`, wiped on reboot). DISA STIG requires a persistent directory: **`/var/log/faillock`**, configured in `/etc/security/faillock.conf`:

```ini
dir = /var/log/faillock
deny = 3
fail_interval = 900
unlock_time = 0
```

Because `/var/log/faillock` is non-default, you **must** pass `--dir /var/log/faillock` when running `faillock`:

```bash
# Check failed login attempts for a user
faillock --dir /var/log/faillock --user jsmith

# Reset / unlock a locked-out user
faillock --dir /var/log/faillock --user jsmith --reset

# View failure records for all users
faillock --dir /var/log/faillock
```

---

## Key User & Group Configuration Files

| File / Directory | Purpose |
| :--- | :--- |
| `/etc/passwd` | Local user accounts (username, UID, primary GID, home dir, login shell). |
| `/etc/shadow` | Local password hashes and password/account aging fields. |
| `/etc/group` | Local groups and their members. |
| `/etc/gshadow` | Secure group passwords and administrators. |
| `/etc/login.defs` | System-wide defaults for `useradd`, UID/GID ranges, `UMASK`, and password aging. |
| `/etc/default/useradd` | Default values used by `useradd` (`SHELL`, `HOME`, `INACTIVE`, `SKEL`). |
| `/etc/skel/` | Skeleton files (`.bashrc`, `.bash_profile`) copied into new user home directories. |
| `/etc/security/pwquality.conf` | Password complexity requirements (STIG). |
| `/etc/security/faillock.conf` | Failed login lockout thresholds and `dir = /var/log/faillock` (STIG). |