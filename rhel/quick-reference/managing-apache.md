---
layout: default
title: Managing Apache HTTP Services
parent: Quick Reference
grand_parent: RHEL
---

# Managing Apache HTTP Services

## Installing Apache

| Task | Command |
|---|---|
| Install Apache HTTP Server | `dnf install httpd` |
| Install SSL/TLS module | `dnf install mod_ssl` |
| Check FIPS mode status | `fips-mode-setup --check` |

> [!NOTE]
> If FIPS mode is enabled, TLS configuration must align with the system cryptographic policy and organizational requirements. Verify FIPS mode before configuring certificates, ciphers, or TLS protocols. RHEL provides TLS configuration through the `mod_ssl` package and its associated configuration files. 

## Apache Service Management

| Task | Command |
|---|---|
| Enable and start Apache | `systemctl enable --now httpd` |
| Check Apache service status | `systemctl status httpd` |
| Restart Apache | `systemctl restart httpd` |

## Configuration Files and Directories

> [!NOTE]
> Apache configuration paths are relative to the `ServerRoot`, typically `/etc/httpd`.

| Location | Purpose |
|---|---|
| `/etc/httpd/conf/httpd.conf` | Main Apache configuration file. |
| `/etc/httpd/conf.d/` | Drop-in configuration files, typically ending in `.conf`. |
| `/etc/httpd/conf.modules.d/` | Module-specific configuration files. |
| `/var/www/html` | Default `DocumentRoot` location. |

RHEL documents `/etc/httpd/conf/httpd.conf` as the main configuration file, with `/etc/httpd/conf.d/` and `/etc/httpd/conf.modules.d/` used for included configuration content. 

| Task | Command |
|---|---|
| View configuration without comment lines | `grep -v '#' /etc/httpd/conf/httpd.conf` |
| Test configuration syntax | `httpd -t` |
| Show virtual-host configuration | `httpd -S` |
| List loaded modules | `httpd -M` |

## Key Configuration Parameters

| Parameter | Purpose |
|---|---|
| `ServerRoot "/etc/httpd"` | Base directory for relative Apache configuration paths. |
| `DocumentRoot "/var/www/html"` | Directory from which Apache serves web content. |
| `Listen 80` | Port on which Apache listens; typically `80` for HTTP and `443` for HTTPS. |
| `DirectoryIndex index.html` | Default file served when a directory is requested. |
| `User apache` / `Group apache` | System account under which Apache worker processes run. |

## Common Apache Configuration Blocks

| Block | Purpose | Example |
|---|---|---|
| `<VirtualHost>` | Defines separate sites using different IP addresses, ports, or hostnames. | `<VirtualHost 192.168.1.10:443>` |
| `<Directory>` | Applies directives to file-system directories and controls access. | `<Directory /var/www/html>` |
| `<IfModule>` | Applies configuration conditionally when a module is loaded. | `<IfModule mod_ssl.c>` |
| `<Files>` | Applies directives to specific files. | `<Files ".htaccess">` |
| `<Location>` | Applies directives to URL paths rather than file-system paths. | `<Location /admin>` |

Commonly used blocks:

- `<VirtualHost>` — Defines individual HTTP or TLS-enabled sites.
- `<Directory>` — Controls file-system access, often within a virtual host.
- `<IfModule>` — Enables conditional configuration based on loaded modules.

## SSL Certificate Generation

### Generate a Private Key

```bash
openssl genrsa -out /etc/pki/tls/private/server.key 2048
```

### Create a Certificate Signing Request with SANs

Create an OpenSSL configuration file:

```bash
vim /tmp/server-san.cnf
```

Generate the CSR:

```bash
openssl req -new \
  -key /etc/pki/tls/private/server.key \
  -out /tmp/server.csr \
  -config /tmp/server-san.cnf
```

Verify Subject Alternative Names in the CSR:

```bash
openssl req -text -noout -in /tmp/server.csr | grep -A 3 "Subject Alternative"
```

Submit the CSR to the organization's internal certificate authority using the approved process.

### Install Certificate Files

```bash
# Install the signed server certificate
cp server.crt /etc/pki/tls/certs/server.crt

# Install the issuing CA chain or bundle
cp ca-bundle.crt /etc/pki/tls/certs/ca-bundle.crt
```

### Example SAN Configuration

```ini
# /tmp/server-san.cnf

[req]
default_bits = 2048
prompt = no
default_md = sha256
distinguished_name = dn
req_extensions = req_ext

[dn]
C = US
ST = Ohio
L = Fairborn
O = Your Organization
OU = IT
CN = secure.example.com

[req_ext]
subjectAltName = @alt_names

[alt_names]
DNS.1 = secure.example.com
DNS.2 = secure
IP.1 = 192.168.1.10
```

RHEL TLS guidance uses `mod_ssl`, a server certificate, its private key, and optional Subject Alternative Names for additional hostnames. 

## Certificate File Permissions

| Task | Command |
|---|---|
| Restrict private key permissions | `chmod 600 /etc/pki/tls/private/server.key` |
| Set server certificate permissions | `chmod 644 /etc/pki/tls/certs/server.crt` |
| Set CA certificate permissions | `chmod 644 /etc/pki/tls/certs/ca-bundle.crt` |
| Set private-key ownership | `chown root:root /etc/pki/tls/private/server.key` |
| Set certificate ownership | `chown root:root /etc/pki/tls/certs/*.crt` |

RHEL documentation uses `root:root` ownership and restrictive permissions for Apache private-key files. 

## Virtual Hosts

### Create a Virtual Host

```bash
# Create virtual-host configuration
vim /etc/httpd/conf.d/vhost.conf

# Create custom DocumentRoot
mkdir -p /var/www/site1

# Set ownership and permissions
chown root:root /var/www/site1
chmod 755 /var/www/site1
chmod 644 /var/www/site1/index.html
```

### DocumentRoot Permissions

For static web content:

- Directories should generally be readable and traversable by Apache, such as mode `755`.
- Files should generally be readable by Apache, such as mode `644`.
- Content can be owned by `root:root` if it is readable by the Apache process.
- Content under `/var/www/` normally receives the `httpd_sys_content_t` SELinux type. 

For a directory that Apache must write to, such as an upload directory:

```bash
chown apache:apache /var/www/site1/uploads
chmod 755 /var/www/site1/uploads
```

> [!WARNING]
> Limit Apache write access to only the specific directories that require it. Do not make the entire `DocumentRoot` writable by Apache unless required by the application design.

### Virtual Host Configuration Parameters

| Parameter | Description |
|---|---|
| `ServerName` | Primary hostname for the virtual host. |
| `DocumentRoot` | Location of web content for the virtual host. |
| `ErrorLog` | Location of the error log. |
| `CustomLog` | Location and format of the access log. |

## SSL/TLS Configuration Parameters

| Parameter | Example Setting | Purpose |
|---|---|---|
| `SSLEngine` | `on` | Enable TLS for the virtual host. |
| `SSLProtocol` | `-all +TLSv1.2 +TLSv1.3` | Restrict allowed TLS versions. |
| `SSLCipherSuite` | `FIPS:!aNULL:!eNULL` | Example cipher-suite configuration; validate against current organizational and system crypto policy. |
| `SSLCertificateFile` | `/etc/pki/tls/certs/server.crt` | Server certificate location. |
| `SSLCertificateKeyFile` | `/etc/pki/tls/private/server.key` | Server private-key location. |
| `SSLCACertificateFile` | `/etc/pki/tls/certs/ca-bundle.crt` | CA chain certificate location. |

> [!IMPORTANT]
> Do not assume that a fixed cipher-suite string is valid for every RHEL release, FIPS configuration, or organizational baseline. Validate `SSLProtocol` and `SSLCipherSuite` settings against the current RHEL system-wide cryptographic policy and applicable security guidance. 

## HTTPS Virtual Host Example

> [!NOTE]
> Name-based virtual hosts can support TLS through Server Name Indication (SNI). Use IP-based virtual hosts only when required by the applicable architecture, client compatibility requirements, or organizational baseline.

```apache
<VirtualHost 192.168.1.10:443>
    ServerName secure.example.com
    DocumentRoot /var/www/secure

    # Directory access control for the custom DocumentRoot
    <Directory /var/www/secure>
        Require all granted
        AllowOverride None
        Options FollowSymLinks
    </Directory>

    # TLS configuration
    SSLEngine on
    SSLProtocol -all +TLSv1.2 +TLSv1.3
    SSLCipherSuite FIPS:!aNULL:!eNULL

    # Certificate files
    SSLCertificateFile /etc/pki/tls/certs/secure.crt
    SSLCertificateKeyFile /etc/pki/tls/private/secure.key
    SSLCACertificateFile /etc/pki/tls/certs/ca-bundle.crt

    # Logs
    ErrorLog logs/secure-error_log
    CustomLog logs/secure-access_log combined
</VirtualHost>
```

## Directory Access Control Directives

Directory blocks control which file-system content Apache may serve and which features are allowed.

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

## SELinux Context for Apache

Apply a persistent SELinux type to a custom `DocumentRoot`:

```bash
# Add persistent context rule
semanage fcontext -a -t httpd_sys_content_t "/var/www/site1(/.*)?"

# Apply the policy-defined context
restorecon -Rv /var/www/site1
```

Temporarily test whether SELinux is causing an issue:

```bash
# Temporary troubleshooting only
setenforce 0

# Restore enforcing mode after testing
setenforce 1
```

Custom Apache content directories require an appropriate SELinux context, such as `httpd_sys_content_t`, for Apache to read the content. 

## Apache Logs

| Log | Location |
|---|---|
| Error log | `/var/log/httpd/error_log` |
| Access log | `/var/log/httpd/access_log` |

Monitor the error log:

```bash
tail -f /var/log/httpd/error_log
```

## Testing Apache

| Task | Command |
|---|---|
| Test local HTTP | `curl http://localhost` |
| Test local HTTPS without certificate validation | `curl -k https://localhost` |
| Test HTTPS with certificate validation | `curl https://secure.example.com` |
| Check listening ports | `ss -tlnp \| grep httpd` |

## Firewall Configuration

### Allow HTTP

```bash
firewall-cmd --permanent --add-service=http
firewall-cmd --reload
```

### Allow HTTPS

```bash
firewall-cmd --permanent --add-service=https
firewall-cmd --reload
```

### Verify Firewall Rules

```bash
firewall-cmd --list-services
```

## Troubleshooting

| Task | Command |
|---|---|
| Check Apache version | `httpd -v` |
| Test configuration syntax | `httpd -t` |
| Show virtual-host configuration | `httpd -S` |
| Check service status | `systemctl status httpd` |
| Check Apache processes | `ps aux \| grep httpd` |
| View certificate details from a server | `openssl s_client -connect server:443 -showcerts` |

## PHP Troubleshooting

| Task | Command |
|---|---|
| Check PHP CLI version | `php -v` |

## Common Troubleshooting Workflow

1. Check Apache service status:

   ```bash
   systemctl status httpd
   ```

2. Test Apache configuration syntax:

   ```bash
   httpd -t
   ```

3. Verify virtual-host configuration:

   ```bash
   httpd -S
   ```

4. Temporarily test whether SELinux is involved:

   ```bash
   setenforce 0
   ```

5. Verify `DocumentRoot` permissions and SELinux context:

   ```bash
   ls -ldZ /var/www/site1
   ```

6. Review Apache error logs:

   ```bash
   tail -f /var/log/httpd/error_log
   ```

7. Check firewall configuration:

   ```bash
   firewall-cmd --list-all
   ```

8. Restore enforcing mode if it was changed for troubleshooting:

   ```bash
   setenforce 1
   ```

## Default Virtual Host Behavior

When multiple virtual hosts are configured, a request without a matching `ServerName` is handled by the first applicable virtual host loaded by Apache. In RHEL configurations, file ordering under `/etc/httpd/conf.d/` can therefore affect which virtual host becomes the default.