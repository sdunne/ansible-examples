# Apache SSL Installation with Self-Signed Certificate

This Ansible playbook installs Apache (httpd) on RHEL 9 with a self-signed SSL certificate.

## Directory Structure

```
.
├── install_apache_ssl.yml    # Main playbook
├── hosts                      # Inventory file
├── templates/
│   ├── ssl-vhost.conf.j2     # SSL virtual host configuration
│   └── index.html.j2         # Sample homepage
└── README.md
```

## Prerequisites

- RHEL 9 target server(s)
- Ansible installed on the control node
- SSH access to target server(s)
- Sudo/root privileges on target server(s)

## What This Playbook Does

1. Installs Apache (httpd), mod_ssl, and openssl packages
2. Generates a self-signed SSL certificate (valid for 365 days)
3. Configures HTTP to HTTPS redirect (port 80 → 443)
4. Configures SSL virtual host on port 443 with HSTS security headers
5. Creates a sample index.html page
6. Configures firewall rules for HTTP/HTTPS (if firewalld is running)
7. Enables and starts the Apache service

## Security Features

- **Automatic HTTPS redirect:** All HTTP traffic is automatically redirected to HTTPS using a 301 permanent redirect
- **HSTS (HTTP Strict Transport Security):** Browsers are instructed to only use HTTPS for future visits (1 year duration)
- **Security headers:** Additional headers protect against clickjacking and MIME-type sniffing attacks
- **Encrypted traffic:** All website traffic is encrypted using TLS/SSL

## Usage

1. **Edit the inventory file** (`hosts`):
   ```ini
   [webservers]
   rhel9-server ansible_host=192.168.1.100 ansible_user=your_user
   ```

2. **Customize SSL certificate variables** (optional) in the playbook:
   - `cert_country`: Country code (default: US)
   - `cert_state`: State/Province
   - `cert_locality`: City
   - `cert_organization`: Organization name
   - `cert_validity_days`: Certificate validity in days (default: 365)

3. **Run the playbook**:
   ```bash
   ansible-playbook -i hosts install_apache_ssl.yml
   ```

   If you need to provide a sudo password:
   ```bash
   ansible-playbook -i hosts install_apache_ssl.yml --ask-become-pass
   ```

## Accessing Your Site

After successful deployment:
- HTTP: `http://your-server-ip`
- HTTPS: `https://your-server-ip`

**Note**: Since this uses a self-signed certificate, your browser will show a security warning. This is expected and you'll need to accept the certificate to proceed.

## Certificate Locations

- Certificate: `/etc/pki/tls/certs/apache-selfsigned.crt`
- Private Key: `/etc/pki/tls/private/apache-selfsigned.key`

## Customization

To use your own certificate files instead of generating a self-signed one, modify the playbook to copy your existing certificate and key files to the paths above.

## Testing

### Test HTTP to HTTPS redirect:
```bash
curl -I http://your-server-ip
# Should show: HTTP/1.1 301 Moved Permanently
# Location: https://your-server-ip/
```

### Verify HSTS header:
```bash
curl -Ik https://your-server-ip
# Should include: Strict-Transport-Security: max-age=31536000; includeSubDomains
```

### Test SSL connection:
```bash
curl -k https://your-server-ip
openssl s_client -connect your-server-ip:443
```

### Browser test:
1. Visit `http://your-server-ip` (should auto-redirect to HTTPS)
2. Open browser DevTools → Network tab
3. Check response headers for HSTS and security headers
4. After first visit, browser should automatically use HTTPS
