# Apache HTTPS Setup Guide

This playbook installs Apache web server with HTTPS configuration using Let's Encrypt certificates on test.sdunne.net (RHEL 9).

## Prerequisites

1. **Certificates Generated**: Run the `acme_letsencrypt.yml` playbook first to obtain certificates:
   ```bash
   ansible-navigator run acme_letsencrypt.yml -m stdout
   ```

2. **SSH Access**: Ensure you have SSH access to test.sdunne.net
   - The playbook uses `ec2-user` with SSH key authentication
   - SSH key: `~/.ssh/apdtempkey.pem` (must be present on the control node)

3. **Sudo/Root Privileges**: The playbook requires root access on the target server (ec2-user should have sudo privileges)

## Usage

This guide uses `ansible-navigator` to run playbooks. Ansible Navigator provides a text-based user interface (TUI) and can use execution environments for consistent runs.

**Inventory Configuration:**
The `hosts` file is referenced in `ansible.cfg`, so you don't need to specify `-i hosts` in your commands.

**Common ansible-navigator options:**
- `-m stdout` - Output mode (stdout for command-line output, interactive for TUI)
- `--ee false` - Disable execution environment (use local Ansible installation)
- `-v`, `-vv`, `-vvv` - Verbose output levels

### Run the Playbook

```bash
# Standard run (inventory is auto-loaded from ansible.cfg)
ansible-navigator run install_apache_https.yml -m stdout

# With verbose output
ansible-navigator run install_apache_https.yml -m stdout -v

# Interactive mode (TUI)
ansible-navigator run install_apache_https.yml

# Disable execution environment (use local Ansible)
ansible-navigator run install_apache_https.yml -m stdout --ee false
```

### What the Playbook Does

1. **Installs packages**: Apache (`httpd`), `mod_ssl`, and `firewalld` on RHEL 9
2. **Creates document root** at `/var/www/test.sdunne.net`
3. **Deploys sample HTML** (from `templates/index.html.j2`) with a modern, responsive design
4. **Copies SSL certificates** from `./certs/` to the server:
   - Certificate: `/etc/pki/tls/certs/test.sdunne.net-fullchain.crt`
   - Private Key: `/etc/pki/tls/private/test.sdunne.net.key`
   - Chain: `/etc/pki/tls/certs/test.sdunne.net-chain.crt`
5. **Disables default SSL configuration** (to prevent conflicts)
6. **Creates SSL Listen configuration** (`00-ssl-listen.conf` to enable port 443)
7. **Configures VirtualHosts**:
   - HTTP (port 80): Redirects to HTTPS
   - HTTPS (port 443): Serves content with SSL/TLS
8. **Enables and starts firewalld**
9. **Opens firewall ports** for HTTP (80) and HTTPS (443)
10. **Starts Apache** and enables it on boot

## Testing

After running the playbook:

```bash
# Test HTTP redirect
curl -I http://test.sdunne.net

# Test HTTPS (with staging cert, use -k to ignore cert warning)
curl -k https://test.sdunne.net

# Test SSL certificate
openssl s_client -connect test.sdunne.net:443 -servername test.sdunne.net
```

## Accessing the Site

- **HTTPS**: https://test.sdunne.net
- **HTTP**: http://test.sdunne.net (auto-redirects to HTTPS)

## Important Notes

### Staging vs Production Certificates

The default configuration uses **staging certificates** from Let's Encrypt, which will show a browser warning.

**For production use:**
1. Update `vars.yml` to use the production ACME directory:
   ```yaml
   acme_directory: https://acme-v02.api.letsencrypt.org/directory
   ```
2. Re-run the certificate generation:
   ```bash
   ansible-navigator run acme_letsencrypt.yml -m stdout
   ```
3. Re-run the Apache setup:
   ```bash
   ansible-navigator run install_apache_https.yml -m stdout
   ```

## Certificate Renewal

Let's Encrypt certificates expire after 90 days. To renew:

1. Run the certificate generation playbook again:
   ```bash
   ansible-navigator run acme_letsencrypt.yml -m stdout
   ```

2. Re-deploy certificates to the web server:
   ```bash
   ansible-navigator run install_apache_https.yml -m stdout
   ```

Or create a cron job to automate this process.

## File Locations on Server

- **Configuration**: 
  - `/etc/httpd/conf.d/test.sdunne.net.conf` - HTTP VirtualHost (redirect)
  - `/etc/httpd/conf.d/test.sdunne.net-ssl.conf` - HTTPS VirtualHost
  - `/etc/httpd/conf.d/00-ssl-listen.conf` - SSL Listen directive for port 443
- **Document Root**: `/var/www/test.sdunne.net/`
- **SSL Certificates**: `/etc/pki/tls/certs/`
- **Private Keys**: `/etc/pki/tls/private/`
- **Logs**: `/var/log/httpd/test.sdunne.net-*.log`

**Note:** The default `/etc/httpd/conf.d/ssl.conf` is renamed to `ssl.conf.disabled` to prevent conflicts. The `00-ssl-listen.conf` file provides the necessary `Listen 443` directive.

## Customization

### Web Content

To customize the website content, edit the `templates/index.html.j2` template file. This is a Jinja2 template that supports Ansible variables:
- `{{ domain_name }}` - The domain name from the playbook
- `{{ ansible_date_time.iso8601 }}` - The deployment timestamp

### Playbook Variables

To modify the playbook for different domains or configurations, edit these variables in `install_apache_https.yml`:

```yaml
vars:
  domain_name: test.sdunne.net           # Change to your domain
  cert_source_dir: "./certs"             # Local cert directory
  cert_dest_dir: "/etc/pki/tls/certs"   # Remote cert location
  key_dest_dir: "/etc/pki/tls/private"  # Remote key location
  document_root: "/var/www/{{ domain_name }}"
```

## Troubleshooting

### Check Apache Status
```bash
ansible-navigator exec -m stdout -- test.sdunne.net -m shell -a "systemctl status httpd"
```

### Check Apache Logs
```bash
ansible-navigator exec -m stdout -- test.sdunne.net -m shell -a "tail -50 /var/log/httpd/test.sdunne.net-ssl-error.log"
```

### Verify Certificate
```bash
ansible-navigator exec -m stdout -- test.sdunne.net -m shell -a "openssl x509 -in /etc/pki/tls/certs/test.sdunne.net-fullchain.crt -text -noout"
```

### Test Configuration
```bash
ansible-navigator exec -m stdout -- test.sdunne.net -m shell -a "httpd -t"
```

### Check Firewall
```bash
ansible-navigator exec -m stdout -- test.sdunne.net -m shell -a "firewall-cmd --list-all"
```
