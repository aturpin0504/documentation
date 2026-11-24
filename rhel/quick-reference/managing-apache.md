---
layout: default
title: Apache HTTP
parent: Quick Reference
grand_parent: RHEL
nav_order: 16
---

# Managing Apache HTTP Services

## Installing and Managing Apache

```bash
# Install Apache HTTP Server and SSL/TLS module
dnf install httpd
dnf install mod_ssl

# Check FIPS mode status
fips-mode-setup --check

# Enable, start, check, or restart Apache
systemctl enable --now httpd
systemctl status httpd
systemctl restart httpd
```

{: .note }
> If FIPS mode is enabled, TLS configuration must align with the system cryptographic policy and organizational requirements. Verify FIPS mode before configuring certificates, ciphers, or TLS protocols.

## Configuration Files and Directories

{: .note }
> Apache configuration paths are relative to the `ServerRoot`, typically `/etc/httpd`.

| Location | Purpose |
|---|---|
| `/etc/httpd/conf/httpd.conf` | Main Apache configuration file. |
| `/etc/httpd/conf.d/` | Drop-in configuration files, typically ending in `.conf`. |
| `/etc/httpd/conf.modules.d/` | Module-specific configuration files. |
| `/var/www/html` | Default `DocumentRoot` location. |

```bash
# View configuration without comment lines
grep -v '#' /etc/httpd/conf/httpd.conf

# Test configuration syntax
httpd -t

# Show virtual-host configuration
httpd -S

# List loaded modules
httpd -M
```

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

## SSL Certificate Generation

### Generate Key, CSR with SANs, and Install Certificates

```bash
# Generate a 2048-bit private key
openssl genrsa -out /etc/pki/tls/private/server.key 2048

# Create an OpenSSL SAN config and generate the CSR
vim /tmp/server-san.cnf
openssl req -new \
  -key /etc/pki/tls/private/server.key \
  -out /tmp/server.csr \
  -config /tmp/server-san.cnf

# Verify Subject Alternative Names in the CSR
openssl req -text -noout -in /tmp/server.csr | grep -A 3 "Subject Alternative"

# Install the signed certificate and CA chain
cp server.crt /etc/pki/tls/certs/server.crt
cp ca-bundle.crt /etc/pki/tls/certs/ca-bundle.crt

# Set ownership and restrictive permissions
chown root:root /etc/pki/tls/private/server.key
chmod 600 /etc/pki/tls/private/server.key
chown root:root /etc/pki/tls/certs/*.crt
chmod 644 /etc/pki/tls/certs/server.crt
chmod 644 /etc/pki/tls/certs/ca-bundle.crt
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

## Virtual Hosts

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

{: .important }
> Do not assume that a fixed cipher-suite string is valid for every RHEL release, FIPS configuration, or organizational baseline. Validate `SSLProtocol` and `SSLCipherSuite` settings against the current RHEL system-wide cryptographic policy and applicable security guidance.

## HTTPS Virtual Host Example

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

Custom Apache content directories require an appropriate SELinux context, such as `httpd_sys_content_t`, for Apache to read the content:

```bash
# Add persistent context rule and apply it
semanage fcontext -a -t httpd_sys_content_t "/var/www/site1(/.*)?"
restorecon -Rv /var/www/site1
```

## Apache Logs and Firewall

| Log | Location |
|---|---|
| Error log | `/var/log/httpd/error_log` |
| Access log | `/var/log/httpd/access_log` |

```bash
# Open HTTP and HTTPS in firewalld
firewall-cmd --permanent --add-service=http
firewall-cmd --permanent --add-service=https
firewall-cmd --reload
firewall-cmd --list-services
```

## Testing and Troubleshooting Apache

```bash
# Check Apache version, syntax, virtual hosts, and modules
httpd -v
httpd -t
httpd -S
httpd -M

# Check service status and processes
systemctl status httpd
ps aux | grep httpd

# Test local HTTP and HTTPS endpoints
curl http://localhost
curl -k https://localhost
curl https://secure.example.com

# Check listening ports and firewall
ss -tlnp | grep httpd
firewall-cmd --list-all

# Verify DocumentRoot permissions and SELinux context
ls -ldZ /var/www/site1

# Monitor the error log
tail -f /var/log/httpd/error_log

# Inspect a server's TLS certificate chain
openssl s_client -connect server:443 -showcerts

# Check PHP CLI version if PHP is used
php -v
```

### Temporarily Testing SELinux Involvement

```bash
# Temporarily switch to permissive mode to test, then restore enforcing mode
setenforce 0
# ... test the site ...
setenforce 1
```

## Default Virtual Host Behavior

When multiple virtual hosts are configured, a request without a matching `ServerName` is handled by the first applicable virtual host loaded by Apache. File ordering under `/etc/httpd/conf.d/` can therefore affect which virtual host becomes the default.