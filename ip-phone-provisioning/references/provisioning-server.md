# Provisioning Server Setup

## Table of Contents

1. [Server Options](#server-options)
2. [HTTP/HTTPS Server (Recommended)](#http-server)
3. [TFTP Server](#tftp-server)
4. [DHCP Configuration](#dhcp-configuration)
5. [Security](#security)
6. [Multi-Vendor Hosting](#multi-vendor)
7. [Dynamic Provisioning](#dynamic-provisioning)
8. [Nginx Configuration](#nginx)
9. [Apache Configuration](#apache)

---

## Server Options

| Protocol | Pros | Cons | Use Case |
|----------|------|------|----------|
| HTTP | Simple, fast, widely supported | No encryption | Lab/internal only |
| HTTPS | Encrypted, recommended | Cert management | Production (preferred) |
| TFTP | Very simple, no auth | No encryption, unreliable, slow | Legacy, small deployments |
| FTP | Auth supported | Plaintext passwords, legacy | Avoid in new deployments |

**Recommendation**: Always use HTTPS for production. Provisioning configs contain SIP
passwords in plaintext — these MUST be encrypted in transit.

---

## HTTP/HTTPS Server (Recommended)

Any standard web server works. Nginx and Apache are the most common.

### Requirements

- Serve static files (XML, CFG, firmware binaries)
- Support HTTP GET requests
- Optional: HTTP Basic Auth for additional security
- Optional: Dynamic config generation via PHP/Python/Node

### Directory Permissions

```bash
# Create provisioning directory
sudo mkdir -p /var/www/provisioning
sudo chown -R www-data:www-data /var/www/provisioning
sudo chmod -R 755 /var/www/provisioning

# Config files should be readable but not listable
# Disable directory listing in web server config
```

---

## TFTP Server

For simple deployments or legacy phones that don't support HTTP well.

### Linux (tftpd-hpa)

```bash
# Install
sudo apt install tftpd-hpa

# Configure /etc/default/tftpd-hpa
TFTP_USERNAME="tftp"
TFTP_DIRECTORY="/var/lib/tftpboot"
TFTP_ADDRESS=":69"
TFTP_OPTIONS="--secure --create"

# Restart
sudo systemctl restart tftpd-hpa

# Set permissions
sudo chown -R tftp:tftp /var/lib/tftpboot
sudo chmod -R 755 /var/lib/tftpboot
```

### MikroTik as TFTP Server

MikroTik routers can serve as a basic TFTP server for small deployments:

```routeros
/ip tftp
add real-filename=flash/provisioning/ req-filename=.*
# This serves all files from flash/provisioning/ via TFTP
```

---

## DHCP Configuration

### MikroTik DHCP Options

```routeros
# Option 66: Provisioning server URL
/ip dhcp-server option
add code=66 name=tftp-server value="'http://10.0.100.1/provisioning/'"

# Option 67: Boot file name (optional, vendor-specific)
/ip dhcp-server option
add code=67 name=boot-file value="'y000000000000.boot'"

# Apply to voice VLAN DHCP server
/ip dhcp-server network
set [find address=10.0.100.0/24] dhcp-option=tftp-server,boot-file

# Or for HTTPS provisioning
/ip dhcp-server option
add code=66 name=prov-url value="'https://prov.athena.net.au/phones/'"
```

### ISC DHCP Server (Linux)

```
# /etc/dhcp/dhcpd.conf
subnet 10.0.100.0 netmask 255.255.255.0 {
    range 10.0.100.10 10.0.100.250;
    option routers 10.0.100.1;
    option domain-name-servers 10.0.100.1;
    
    # Provisioning server URL
    option tftp-server-name "http://10.0.100.1/provisioning/";
    
    # Or with option 160 for some vendors
    option option-160 "https://prov.athena.net.au/phones/";
}

# Vendor-specific DHCP classes (optional but clean)
class "YealinkPhones" {
    match if substring(option vendor-class-identifier, 0, 7) = "yealink";
    option tftp-server-name "http://10.0.100.1/provisioning/yealink/";
}

class "PolycomPhones" {
    match if substring(option vendor-class-identifier, 0, 13) = "Polycom-VVX";
    option tftp-server-name "http://10.0.100.1/provisioning/polycom/";
}

class "SnomPhones" {
    match if substring(option vendor-class-identifier, 0, 4) = "snom";
    option tftp-server-name "http://10.0.100.1/provisioning/snom/";
}
```

### Kea DHCP Server

```json
{
  "Dhcp4": {
    "subnet4": [{
      "subnet": "10.0.100.0/24",
      "pools": [{"pool": "10.0.100.10-10.0.100.250"}],
      "option-data": [
        {"name": "routers", "data": "10.0.100.1"},
        {"name": "tftp-server-name", "data": "http://10.0.100.1/provisioning/"}
      ]
    }]
  }
}
```

---

## Security

### HTTPS with Let's Encrypt

```bash
# Install certbot
sudo apt install certbot python3-certbot-nginx

# Get certificate
sudo certbot --nginx -d prov.athena.net.au

# Auto-renewal (certbot adds a cron automatically)
sudo certbot renew --dry-run
```

### HTTP Basic Auth (Additional Layer)

```bash
# Create password file
sudo htpasswd -c /etc/nginx/.htpasswd provision_user

# Nginx config
location /provisioning/ {
    auth_basic "Phone Provisioning";
    auth_basic_user_file /etc/nginx/.htpasswd;
}
```

Phone config for HTTP auth:

- **Yealink**: `auto_provision.server.url = https://user:pass@prov.example.com/yealink/`
- **Polycom**: Set `prov.login.user` and `prov.login.password`
- **Snom**: Embed in URL: `https://user:pass@prov.example.com/snom/`
- **Fanvil**: Set username/password in Auto Provision settings

### Config File Encryption

Some vendors support encrypting config files so they can't be read even if intercepted:

- **Yealink**: AES encryption with `auto_provision.aes_key_in_file`
- **Polycom**: Config file encryption (keys in provisioning settings)
- **Fanvil**: Config Encryption Key in Auto Provision settings

### IP Restrictions

Lock down the provisioning server to your known phone subnets:

```nginx
# Nginx - restrict to voice VLAN subnets only
location /provisioning/ {
    allow 10.0.100.0/24;     # Voice VLAN site A
    allow 10.0.200.0/24;     # Voice VLAN site B
    allow 203.0.113.0/24;    # Public IP range (for remote phones)
    deny all;
}
```

---

## Multi-Vendor Hosting

When hosting configs for multiple vendors on one server, separate by vendor at the top level:

```
/var/www/provisioning/
├── yealink/      ← DHCP Option 66 for Yealink: http://prov/provisioning/yealink/
├── polycom/      ← DHCP Option 66 for Polycom: http://prov/provisioning/polycom/
├── snom/         ← DHCP Option 66 for Snom: http://prov/provisioning/snom/
└── fanvil/       ← DHCP Option 66 for Fanvil: http://prov/provisioning/fanvil/
```

If you can't use vendor-specific DHCP classes (single DHCP scope for all vendors), some
approaches:

1. **Vendor-specific DHCP classes** (shown above) — cleanest but requires DHCP vendor matching
2. **Redirect server** — Single URL that inspects User-Agent and redirects to vendor-specific path
3. **Universal directory** — Put all vendor configs in one directory (works but messy)
4. **Cloud redirect services** — Each vendor's RPS/ZTP/SRAPS/FDPS points to vendor-specific URL

---

## Dynamic Provisioning

For larger deployments, generate configs dynamically from a database instead of static files.

### PHP Example (Yealink)

```php
<?php
// /var/www/provisioning/yealink/config.php
// Accessed as: http://prov/provisioning/yealink/config.php?mac=001565abcdef

$mac = preg_replace('/[^a-f0-9]/', '', strtolower($_GET['mac'] ?? ''));
if (strlen($mac) !== 12) { http_response_code(404); exit; }

// Look up phone in database
$db = new PDO('mysql:host=localhost;dbname=provisioning', 'user', 'pass');
$stmt = $db->prepare('SELECT * FROM phones WHERE mac = ?');
$stmt->execute([$mac]);
$phone = $stmt->fetch(PDO::FETCH_ASSOC);
if (!$phone) { http_response_code(404); exit; }

header('Content-Type: text/xml');
echo '<?xml version="1.0" encoding="utf-8"?>' . "\n";
?>
<y0000000000xx>
  <account.1.enable>1</account.1.enable>
  <account.1.label><?= htmlspecialchars($phone['extension']) ?></account.1.label>
  <account.1.display_name><?= htmlspecialchars($phone['display_name']) ?></account.1.display_name>
  <account.1.auth_name><?= htmlspecialchars($phone['auth_user']) ?></account.1.auth_name>
  <account.1.user_name><?= htmlspecialchars($phone['extension']) ?></account.1.user_name>
  <account.1.password><?= htmlspecialchars($phone['sip_password']) ?></account.1.password>
  <account.1.sip_server.1.address><?= htmlspecialchars($phone['sip_server']) ?></account.1.sip_server.1.address>
  <account.1.sip_server.1.port>5060</account.1.sip_server.1.port>
</y0000000000xx>
```

### Laravel Integration

For Athena Networks-style Laravel backends:

```php
// routes/api.php
Route::get('/provisioning/{vendor}/{filename}', [ProvisioningController::class, 'serve']);

// ProvisioningController.php
public function serve(string $vendor, string $filename)
{
    // Extract MAC from filename
    $mac = Str::before($filename, '.');
    $phone = Phone::where('mac_address', $mac)->firstOrFail();
    
    $template = match ($vendor) {
        'yealink' => 'provisioning.yealink',
        'polycom' => 'provisioning.polycom',
        'snom'    => 'provisioning.snom',
        'fanvil'  => 'provisioning.fanvil',
    };
    
    return response()
        ->view($template, ['phone' => $phone])
        ->header('Content-Type', 'text/xml');
}
```

---

## Nginx Configuration

### Full Nginx Config for Provisioning

```nginx
server {
    listen 80;
    listen 443 ssl;
    server_name prov.athena.net.au;
    
    ssl_certificate /etc/letsencrypt/live/prov.athena.net.au/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/prov.athena.net.au/privkey.pem;
    
    root /var/www/provisioning;
    
    # Disable directory listing
    autoindex off;
    
    # Logging for provisioning requests (useful for debugging)
    access_log /var/log/nginx/provisioning_access.log;
    error_log /var/log/nginx/provisioning_error.log;
    
    # Serve static config files
    location / {
        # IP restriction
        allow 10.0.100.0/24;
        allow 10.0.200.0/24;
        deny all;
        
        # MIME types for phone config files
        types {
            text/xml  xml cfg;
            application/octet-stream  boot rom bin fw;
        }
    }
    
    # Dynamic provisioning (optional)
    location ~ \.php$ {
        fastcgi_pass unix:/var/run/php/php8.2-fpm.sock;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
        include fastcgi_params;
    }
    
    # Firmware files — allow larger downloads
    location ~* \.(rom|bin|fw|ld)$ {
        sendfile on;
        tcp_nopush on;
    }
}
```

---

## Apache Configuration

```apache
<VirtualHost *:80>
    ServerName prov.athena.net.au
    DocumentRoot /var/www/provisioning
    
    <Directory /var/www/provisioning>
        Options -Indexes
        AllowOverride None
        
        # IP restriction
        Require ip 10.0.100.0/24
        Require ip 10.0.200.0/24
    </Directory>
    
    # MIME types
    AddType text/xml .cfg
    AddType application/octet-stream .boot .rom .bin .fw
    
    # Logging
    CustomLog /var/log/apache2/provisioning_access.log combined
    ErrorLog /var/log/apache2/provisioning_error.log
</VirtualHost>
```
