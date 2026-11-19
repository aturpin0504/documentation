# RHEL Quick Reference

A focused daily reference for RHEL administration.

## Topics

- **[Essential Command Line Tools](/rhel/quick-reference/essential-cli.md)** — Help lookup, history, I/O redirection, paging, environment variables, user identity, `su`/`sudo`, `visudo`, and power management.
- **[File System Management](/rhel/quick-reference/filesystem-management.md)** — FHS layout, mounting & STIG `/etc/fstab` options, `ls`/`stat`/`file`, `cp`/`mv -Z` SELinux behavior, `find`, links, and `tar`/`gzip`.
- **[Text Processing](/rhel/quick-reference/text-processing.md)** — `cat`/`head`/`tail`, `cut`, `sort`, `vi` essentials, and when/how to use `grep`, `sed`, and `awk`.
- **[Package & Satellite Management](/rhel/quick-reference/package-management.md)** — Satellite 6.19 registration & manifests, repolist/custom repos, STIG RSA GPG signing, `satellite-maintain`, `dnf history`, module streams, and package search.
- **[User & Group Management](/rhel/quick-reference/users-and-groups.md)** — `useradd`/`usermod`/`userdel`, `wheel` & `nologin`, smart card password aging & PAM `no_pass_expiry`, STIG `pwquality.conf`, `faillock --dir /var/log/faillock`, and key config files.