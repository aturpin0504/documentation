---
layout: default
title: Apache HTTP
parent: Quick Reference
grand_parent: RHEL
nav_order: 16
---

# Managing Apache HTTP Services

## Installation and Service Management

Apache (`httpd`) and `mod_ssl` are not installed by default on RHEL 8+. Install and manage them with `dnf` and `systemctl`:

```bash
# Install Apache HTTP Server and SSL/TLS module
dnf install -y httpd mod_ssl

# Enable, start, check, or restart Apache
systemctl enable --now httpd
systemctl status httpd
systemctl restart httpd
```

---

## Configuration Files, Core Directives, and Blocks

{: .note }
> Apache configuration paths are relative to `ServerRoot`, typically `/etc/httpd`.

| Location | Purpose |
|---|---|
| `/etc/httpd/conf/httpd.conf` | Main Apache configuration file. |
| `/etc/httpd/conf.d/` | Drop-in configuration files, typically ending in `.conf`. |
| `/etc/httpd/conf.modules.d/` | Module-specific configuration files. |
| `/var/www/html` | Default `DocumentRoot` location. |

### Key Configuration Parameters

| Parameter | Purpose |
|---|---|
| `ServerRoot "/etc/httpd"` | Base directory for relative Apache configuration paths. |
| `DocumentRoot "/var/www/html"` | Directory from which Apache serves web content. |
| `Listen 80` | Port on which Apache listens; typically `80` for HTTP and `443` for HTTPS. |
| `DirectoryIndex index.html` | Default file served when a directory is requested. |
| `User apache` / `Group apache` | System account under which Apache worker processes run. |

### Common Configuration Blocks

| Block | Purpose | Example |
|---|---|---|
| `<VirtualHost>` | Defines separate sites using different IP addresses, ports, or hostnames. | `<VirtualHost 192.168.1.10:443>` |
| `<Directory>` | Applies directives to file-system directories and controls access. | `<Directory /var/www/html>` |
| `<IfModule>` | Applies configuration conditionally when a module is loaded. | `<IfModule mod_ssl.c>` |
| `<Files>` | Applies directives to specific files. | `<Files ".htaccess">` |
| `<Location>` | Applies directives to URL paths rather than file-system paths. | `<Location /admin>` |

---

## Virtual Hosts and Directory Access Control

### Creating a Virtual Host

```bash
# Create virtual-host configuration and custom DocumentRoot
vim /etc/httpd/conf.d/vhost.conf
mkdir -p /var/www/site1

# Set ownership and permissions for static content
chown root:root /var/www/site1
chmod 755 /var/www/site1
chmod 644 /var/www/site1/index.html

# For a directory Apache must write to (such as uploads)
chown apache:apache /var/www/site1/uploads
chmod 755 /var/www/site1/uploads
```

{: .warning }
> Limit Apache write access to only the specific directories that require it. Do not make the entire `DocumentRoot` writable by Apache unless required by the application design.

| Parameter | Description |
|---|---|
| `ServerName` | Primary hostname for the virtual host. |
| `DocumentRoot` | Location of web content for the virtual host. |
| `ErrorLog` | Location of the error log. |
| `CustomLog` | Location and format of the access log. |

#### Default Virtual Host Behavior

When multiple virtual hosts are configured, a request without a matching `ServerName` is handled by the first applicable virtual host loaded by Apache. File ordering under `/etc/httpd/conf.d/` can therefore affect which virtual host becomes the default.

{: .note }
> When wrapping virtual-host directives in `<IfModule>`, use either the module identifier from `httpd -M` without an extension (for example, `<IfModule ssl_module>`) or the `.c` source name (`<IfModule mod_ssl.c>`). Do not use `.so` (`mod_ssl.so` will not match).

#### Web Content Ownership and SELinux Context

- **Ownership (`root:root` vs. `apache:apache`)**: Although Apache runs as `apache:apache`, static content and application source code should normally be owned by `root:root` (`755` directories, `644` files) so a compromised web process cannot modify application code. Reserve `apache:apache` ownership (and `httpd_sys_rw_content_t`) for directories Apache or PHP must write to (such as `uploads/`, `cache/`, or `sessions/`).
- **Default context under `/var/www/html/`**: `/var/www/html` and all subdirectories (for example, `/var/www/html/password` and `/var/www/html/phpipam`) are already mapped in RHEL's SELinux policy to `httpd_sys_content_t`, so `semanage fcontext` is not needed for paths under `/var/www/html/`.
- **Copying vs. moving files (`-Z` and `restorecon`)**:
  - `cp` creates new files at the destination and inherits `httpd_sys_content_t`. Plain `mv` (or `cp -a`) preserves the source context (such as `user_tmp_t` from `/tmp`).
  - For single files, `cp -Z` and `mv -Z` set the destination to the SELinux policy default for that path.
  - When moving a whole directory tree on the same filesystem, `mv -Z` only updates the top-level directory—not nested files—so run `restorecon` afterward:

```bash
# Restore policy-defined SELinux contexts recursively after deploying or moving files
restorecon -RFv /var/www/html/password /var/www/html/phpipam
```

### Directory Access Control Directives

| Directive | Options | Purpose |
|---|---|---|
| `Require` | `all granted`, `all denied` | Allow or deny HTTP access to a directory. |
| `AllowOverride` | `None`, `All` | Control whether `.htaccess` files can override configuration. |
| `Options` | `Indexes`, `FollowSymLinks`, `-Indexes` | Control directory listings, symbolic-link following, and related directory behavior. |

Recommended security-oriented settings:

```apache
<Directory /var/www/secure>
    Require all granted
    AllowOverride None
    Options FollowSymLinks -Indexes
</Directory>
```

---

## SELinux and Firewall Configuration

### SELinux Context for Apache

Custom Apache content directories require an appropriate SELinux context, such as `httpd_sys_content_t`, for Apache to read the content:

```bash
# Add persistent context rule and apply it
semanage fcontext -a -t httpd_sys_content_t "/var/www/site1(/.*)?"
restorecon -Rv /var/www/site1
```

### Firewall Rules

```bash
# Open HTTP and HTTPS in firewalld
firewall-cmd --permanent --add-service=http
firewall-cmd --permanent --add-service=https
firewall-cmd --reload
firewall-cmd --list-services
```

---

## TLS and HTTPS Virtual Hosts

{: .note }
> If FIPS mode is enabled (`fips-mode-setup --check`), TLS configuration must align with the system-wide cryptographic policy. Verify FIPS mode before configuring certificates, ciphers, or TLS protocols.

### SSL/TLS Configuration Parameters

| Parameter | Example Setting | Purpose |
|---|---|---|
| `SSLEngine` | `on` | Enable TLS for the virtual host. |
| `SSLProtocol` | `-all +TLSv1.2 +TLSv1.3` | Restrict allowed TLS versions. |
| `SSLCipherSuite` | `PROFILE=SYSTEM` | Inherit the active RHEL system-wide cryptographic policy (`DEFAULT`, `FIPS`, etc.). |
| `SSLCertificateFile` | `/etc/pki/tls/certs/server.crt` | Server certificate location. |
| `SSLCertificateKeyFile` | `/etc/pki/tls/private/server.key` | Server private-key location (`chmod 600`, `root:root`). |
| `SSLCACertificateFile` | `/etc/pki/tls/certs/ca-bundle.crt` | CA chain certificate location. |

{: .important }
> Do not assume that a fixed cipher-suite string is valid for every RHEL release, FIPS configuration, or organizational baseline. Using `SSLCipherSuite PROFILE=SYSTEM` keeps Apache aligned with the active RHEL system-wide cryptographic policy.

### HTTPS Virtual Host Example

{: .note }
> Name-based virtual hosts can support TLS through Server Name Indication (SNI). Use IP-based virtual hosts only when required by the applicable architecture, client compatibility requirements, or organizational baseline.

```apache
<VirtualHost 192.168.1.10:443>
    ServerName secure.example.com
    DocumentRoot /var/www/secure

    # Directory access control for the custom DocumentRoot
    <Directory /var/www/secure>
        Require all granted
        AllowOverride None
        Options FollowSymLinks -Indexes
    </Directory>

    # TLS configuration
    SSLEngine on
    SSLProtocol -all +TLSv1.2 +TLSv1.3
    SSLCipherSuite PROFILE=SYSTEM

    # Certificate files
    SSLCertificateFile /etc/pki/tls/certs/secure.crt
    SSLCertificateKeyFile /etc/pki/tls/private/secure.key
    SSLCACertificateFile /etc/pki/tls/certs/ca-bundle.crt

    # Logs
    ErrorLog logs/secure-error_log
    CustomLog logs/secure-access_log combined
</VirtualHost>
```

---

## Verifying and Troubleshooting

### Configuration and Service Checks

```bash
# View httpd.conf without comment lines
grep -v '#' /etc/httpd/conf/httpd.conf

# Check Apache version, syntax, virtual hosts, and loaded modules
httpd -v
httpd -t
httpd -S
httpd -M

# Check service status, processes, and listening ports
systemctl status httpd
ps aux | grep httpd
ss -tlnp | grep httpd
```

### Verifying FIPS TLS Configuration

On RHEL, Apache (`mod_ssl`) inherits the system-wide cryptographic policy through OpenSSL:

```bash
# Confirm FIPS mode and active system crypto policy
fips-mode-setup --check
update-crypto-policies --show

# Inspect the OpenSSL cipher and protocol settings generated by the policy
cat /etc/crypto-policies/back-ends/openssl.config
cat /etc/crypto-policies/back-ends/opensslcnf.config

# List the individual ciphers allowed under PROFILE=SYSTEM
openssl ciphers -v 'PROFILE=SYSTEM'

# Check active TLS directives in Apache configuration
grep -E '^\s*(SSLProtocol|SSLCipherSuite|SSLHonorCipherOrder)' /etc/httpd/conf.d/*.conf

# Verify certificate key size (>= 2048-bit RSA or approved ECDSA) and signature algorithm (SHA-256+)
openssl x509 -in /etc/pki/tls/certs/server.crt -noout -text | grep -E 'Public-Key:|Signature Algorithm:'
```

Test the live TLS endpoint to confirm only approved protocols and ciphers are negotiated:

```bash
# Verify TLS 1.2 and TLS 1.3 handshakes and inspect the certificate chain
openssl s_client -connect localhost:443 -tls1_2
openssl s_client -connect localhost:443 -tls1_3
openssl s_client -connect server:443 -showcerts

# Confirm deprecated protocols (such as TLS 1.1) are rejected
openssl s_client -connect localhost:443 -tls1_1
```

{: .note }
> On RHEL 9 in FIPS mode, TLS 1.2 connections also require Extended Master Secret (EMS) support from the client; legacy clients without EMS or TLS 1.3 will fail to connect.

### Logs, Endpoints, and SELinux Troubleshooting

| Log | Location |
|---|---|
| Error log | `/var/log/httpd/error_log` |
| Access log | `/var/log/httpd/access_log` |

```bash
# Monitor the error log
tail -f /var/log/httpd/error_log

# Test local HTTP and HTTPS endpoints
curl http://localhost
curl -k https://localhost
curl https://secure.example.com

# Check firewall rules and DocumentRoot permissions/SELinux context
firewall-cmd --list-all
ls -ldZ /var/www/site1

# Check PHP CLI version if PHP is used
php -v
```

#### Temporarily Testing SELinux Involvement

```bash
# Temporarily switch to permissive mode to test, then restore enforcing mode
setenforce 0
# ... test the site ...
setenforce 1
```
