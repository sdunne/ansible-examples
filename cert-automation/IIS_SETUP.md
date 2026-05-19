# IIS HTTPS Setup on Windows Server 2019

This playbook installs and configures Microsoft IIS (Internet Information Services) on Windows Server 2019 with HTTPS support using a self-signed SSL certificate and sample web content.

## Overview

The `install_iis_https.yml` playbook performs the following:

1. **IIS Installation**: Installs IIS Web Server with all necessary features
2. **Feature Installation**: 
   - Common HTTP features (static content, default documents, directory browsing)
   - Health and diagnostics (logging, tracing, request monitoring)
   - Performance features (compression)
   - Security features (authentication, filtering)
   - Application development (.NET 4.5, ASP.NET, ISAPI)
   - Management tools (IIS Manager, scripting tools)
3. **SSL Certificate**: Creates a self-signed SSL certificate for HTTPS
4. **Website Configuration**: Creates a custom website with HTTP and HTTPS bindings
5. **Sample Content**: Deploys sample HTML and CSS files
6. **Firewall Configuration**: Opens HTTP port (80) and HTTPS port (443)
7. **Service Management**: Ensures IIS service (W3SVC) is running

## Prerequisites

### Windows Server Configuration

1. **Windows Server 2019** (or newer)
2. **WinRM enabled** for Ansible connectivity
3. **PowerShell 5.1+** (included by default in Windows Server 2019)

### Ansible Controller Setup

1. **Python packages**:
   ```bash
   pip install pywinrm
   ```

2. **Ansible collections**:
   ```bash
   # Install from requirements file (recommended)
   ansible-galaxy collection install -r requirements.yml

   # Or install individually
   ansible-galaxy collection install microsoft.iis
   ansible-galaxy collection install ansible.windows
   ansible-galaxy collection install community.windows
   ```

   **Note**: The `microsoft.iis` collection is required for IIS management. The deprecated `community.windows` IIS modules are being phased out in favor of `microsoft.iis`.

### Windows Host Configuration

Configure WinRM on the Windows server:

```powershell
# Run on Windows Server as Administrator
# Enable WinRM HTTPS
winrm quickconfig -force

# Set authentication
winrm set winrm/config/service/auth '@{Basic="true"}'
winrm set winrm/config/service '@{AllowUnencrypted="true"}'

# Configure firewall
Enable-NetFirewallRule -DisplayName "Windows Remote Management (HTTP-In)"

# For HTTPS (recommended for production):
# Create self-signed certificate and configure HTTPS listener
$cert = New-SelfSignedCertificate -DnsName "your-server-name" -CertStoreLocation "Cert:\LocalMachine\My"
winrm create winrm/config/Listener?Address=*+Transport=HTTPS "@{Hostname=`"your-server-name`";CertificateThumbprint=`"$($cert.Thumbprint)`"}"
```

## Inventory Configuration

Add your Windows server to the `hosts` file:

```ini
[windows_servers]
win-iis-01 ansible_host=192.168.1.100

[windows_servers:vars]
ansible_user=Administrator
ansible_password=YourPassword
ansible_connection=winrm
ansible_winrm_transport=basic
ansible_winrm_server_cert_validation=ignore
ansible_port=5985

# For HTTPS WinRM (recommended):
# ansible_port=5986
# ansible_winrm_transport=certificate
```

**Security Note**: For production, use Ansible Vault to encrypt credentials:

```bash
ansible-vault create group_vars/windows_servers/vault.yml
```

Add to `vault.yml`:
```yaml
vault_ansible_user: Administrator
vault_ansible_password: YourSecurePassword
```

Reference in inventory:
```ini
[windows_servers:vars]
ansible_user={{ vault_ansible_user }}
ansible_password={{ vault_ansible_password }}
```

## Variables

Default variables in the playbook:

| Variable | Default Value | Description |
|----------|---------------|-------------|
| `iis_domain_name` | `www.example.com` | Domain name for the website and SSL certificate |
| `iis_site_name` | `DefaultWebSite` | IIS website name |
| `http_port` | `80` | HTTP port |
| `https_port` | `443` | HTTPS port |
| `document_root` | `C:\inetpub\wwwroot\{site_name}` | Website root directory |
| `site_binding_ip` | `*` | IP binding (all IPs) |
| `cert_friendly_name` | `{site_name}-SelfSigned` | SSL certificate friendly name |
| `cert_validity_days` | `365` | Certificate validity period in days |

## Usage

### Basic Usage

Run the playbook with default settings:

```bash
ansible-playbook install_iis_https.yml
```

### Custom Variables

Override variables at runtime:

```bash
ansible-playbook install_iis_https.yml \
  -e "iis_domain_name=www.mysite.com" \
  -e "iis_site_name=MySite"
```

### Specific Host

Target a specific Windows server:

```bash
ansible-playbook install_iis_https.yml --limit win-iis-01
```

### With Vault Password

If using Ansible Vault for credentials:

```bash
ansible-playbook install_iis_https.yml --ask-vault-pass
```

### Dry Run (Check Mode)

Preview changes without applying them:

```bash
ansible-playbook install_iis_https.yml --check
```

## Post-Installation

### Working with Self-Signed Certificates

Self-signed certificates will show browser security warnings. Here's how to handle them:

**Browser Warning**:
- Chrome/Edge: Click "Advanced" → "Proceed to [site] (unsafe)"
- Firefox: Click "Advanced" → "Accept the Risk and Continue"

**Trust the Certificate (for testing)**:

On Windows client:
1. Download the certificate from the browser
2. Install it in "Trusted Root Certification Authorities"
3. Or use PowerShell:
   ```powershell
   # Export certificate from server
   $cert = Get-ChildItem Cert:\LocalMachine\My | Where-Object {$_.Subject -like "*your-domain*"}
   Export-Certificate -Cert $cert -FilePath C:\temp\site-cert.cer
   
   # Import to Trusted Root on client
   Import-Certificate -FilePath C:\temp\site-cert.cer -CertStoreLocation Cert:\LocalMachine\Root
   ```

On Linux client:
```bash
# Download certificate
echo -n | openssl s_client -connect your-domain:443 | sed -ne '/-BEGIN CERTIFICATE-/,/-END CERTIFICATE-/p' > site-cert.crt

# Install (Ubuntu/Debian)
sudo cp site-cert.crt /usr/local/share/ca-certificates/
sudo update-ca-certificates

# Or just use -k flag with curl
curl -k https://your-domain
```

**For Production**: Replace the self-signed certificate with one from a trusted CA.

### Verify Installation

1. **From Ansible controller**:
   ```bash
   # HTTPS (ignore self-signed certificate warning)
   curl -k https://your-domain-name
   
   # HTTP
   curl http://your-domain-name
   ```

2. **From Windows server**:
   - Open Internet Explorer/Edge: `https://localhost` (accept certificate warning)
   - Or run PowerShell:
     ```powershell
     # Ignore self-signed certificate
     [System.Net.ServicePointManager]::ServerCertificateValidationCallback = {$true}
     Invoke-WebRequest https://localhost
     ```

3. **IIS Manager**:
   - Run `inetmgr` on the Windows server
   - Or: Server Manager > Tools > IIS Manager
   - View bindings: Select website → Actions → Bindings

4. **View SSL Certificate**:
   ```powershell
   # List certificates in My store
   Get-ChildItem Cert:\LocalMachine\My
   
   # View specific certificate
   certutil -store My <thumbprint>
   ```

### IIS Logs

View IIS logs at:
```
C:\inetpub\logs\LogFiles\W3SVC*
```

### Manage IIS Service

```powershell
# Check service status
Get-Service W3SVC

# Restart IIS
iisreset

# Stop/Start IIS
Stop-Service W3SVC
Start-Service W3SVC
```

## Customization

### Add More Content

1. Place files in a `files/` directory
2. Add a task to copy them:

```yaml
- name: Copy additional content
  ansible.windows.win_copy:
    src: files/myapp/
    dest: "{{ document_root }}\\"
```

### Use a Real SSL Certificate

To replace the self-signed certificate with a real one from a CA (like Let's Encrypt):

1. **Obtain certificate files**:
   - Certificate file (.crt or .cer)
   - Private key (.key)
   - Chain/intermediate certificates

2. **Import certificate to Windows**:
   ```yaml
   - name: Import SSL certificate
     ansible.windows.win_certificate_store:
       path: "{{ cert_file_path }}"
       state: present
       store_location: LocalMachine
       store_name: My
       key_exportable: no
       key_storage: machine
     register: imported_cert
   
   - name: Update IIS website with imported certificate
     microsoft.iis.website:
       name: "{{ site_name }}"
       state: started
       physical_path: "{{ document_root }}"
       bindings:
         - protocol: https
           port: 443
           host_header: "{{ domain_name }}"
           certificate_hash: "{{ imported_cert.thumbprints[0] }}"
           certificate_store_name: My
           ssl_flags: 1
   ```

### Configure HTTP to HTTPS Redirect

To force all HTTP traffic to HTTPS:

```yaml
- name: Install URL Rewrite module
  ansible.windows.win_chocolatey:
    name: urlrewrite
    state: present

- name: Configure HTTP to HTTPS redirect
  ansible.windows.win_powershell:
    script: |
      Import-Module WebAdministration
      $siteName = "{{ site_name }}"
      
      Add-WebConfigurationProperty -Filter "system.webServer/rewrite/rules" `
        -PSPath "IIS:\Sites\$siteName" `
        -Name "." `
        -Value @{
          name='HTTP to HTTPS'
          patternSyntax='Wildcard'
          stopProcessing='True'
        }
      
      Set-WebConfigurationProperty -Filter "system.webServer/rewrite/rules/rule[@name='HTTP to HTTPS']/match" `
        -PSPath "IIS:\Sites\$siteName" `
        -Name "url" `
        -Value "*"
      
      Set-WebConfigurationProperty -Filter "system.webServer/rewrite/rules/rule[@name='HTTP to HTTPS']/action" `
        -PSPath "IIS:\Sites\$siteName" `
        -Name "type" `
        -Value "Redirect"
      
      Set-WebConfigurationProperty -Filter "system.webServer/rewrite/rules/rule[@name='HTTP to HTTPS']/action" `
        -PSPath "IIS:\Sites\$siteName" `
        -Name "url" `
        -Value "https://{HTTP_HOST}/{R:1}"
```

### Extend Certificate Validity

To create a certificate valid for longer:

```bash
ansible-playbook install_iis_https.yml -e "cert_validity_days=730"  # 2 years
```

### Modify Application Pool Settings

```yaml
- name: Configure application pool
  microsoft.iis.web_app_pool:
    name: "{{ site_name }}AppPool"
    state: started
    attributes:
      managedRuntimeVersion: "v4.0"
      managedPipelineMode: "Integrated"
      processModel.identityType: "ApplicationPoolIdentity"
      recycling.periodicRestart.time: "1.05:00:00"  # 29 hours
```

## Troubleshooting

### WinRM Connection Issues

1. **Test WinRM connectivity**:
   ```bash
   ansible windows_servers -m win_ping
   ```

2. **Verify WinRM listener**:
   ```powershell
   # On Windows server
   winrm enumerate winrm/config/Listener
   ```

3. **Check firewall rules**:
   ```powershell
   Get-NetFirewallRule -DisplayName "*WinRM*"
   ```

### IIS Installation Failures

1. **Check Windows Update**: Ensure Windows is up to date
2. **Verify .NET Framework**: .NET 4.5+ should be installed
3. **Review logs**: Check Application Event Viewer on Windows

### Website Not Accessible

1. **Check firewall**:
   ```powershell
   Get-NetFirewallRule -DisplayName "IIS HTTP Traffic"
   Get-NetFirewallRule -DisplayName "IIS HTTPS Traffic"
   ```

2. **Verify IIS service**:
   ```powershell
   Get-Service W3SVC
   ```

3. **Test locally**:
   ```powershell
   Invoke-WebRequest http://localhost
   
   # For HTTPS (ignore cert validation)
   [System.Net.ServicePointManager]::ServerCertificateValidationCallback = {$true}
   Invoke-WebRequest https://localhost
   ```

4. **Check application pool**:
   ```powershell
   Import-Module WebAdministration
   Get-IISAppPool
   ```

5. **Verify bindings**:
   ```powershell
   Import-Module WebAdministration
   Get-WebBinding -Name "DefaultWebSite"
   ```

### SSL/Certificate Issues

1. **Certificate not found**:
   ```powershell
   # List all certificates in My store
   Get-ChildItem Cert:\LocalMachine\My | Format-List Subject, Thumbprint, FriendlyName
   ```

2. **Certificate binding failed**:
   ```powershell
   # Check SSL bindings
   netsh http show sslcert
   
   # View IIS bindings
   Import-Module WebAdministration
   Get-WebBinding -Protocol https
   ```

3. **Recreate self-signed certificate**:
   ```powershell
   # Remove old certificate
   Get-ChildItem Cert:\LocalMachine\My | Where-Object {$_.FriendlyName -eq "YourSite-SelfSigned"} | Remove-Item
   
   # Create new certificate
   $cert = New-SelfSignedCertificate -DnsName "your-domain.com" -CertStoreLocation "Cert:\LocalMachine\My"
   
   # Update IIS binding
   $binding = Get-WebBinding -Name "DefaultWebSite" -Protocol https
   $binding.AddSslCertificate($cert.Thumbprint, "My")
   ```

4. **Test SSL/TLS connection**:
   ```bash
   # From Linux/Mac
   openssl s_client -connect your-domain:443 -showcerts
   
   # Check certificate details
   echo | openssl s_client -connect your-domain:443 2>/dev/null | openssl x509 -noout -text
   ```

## Security Considerations

1. **Replace self-signed certificates in production**: 
   - Self-signed certificates provide encryption but no identity verification
   - Use certificates from trusted CAs (Let's Encrypt, DigiCert, etc.)
   - Consider Azure Key Vault for certificate management in Azure environments

2. **SSL/TLS Configuration**:
   - Disable older protocols (SSL 2.0, SSL 3.0, TLS 1.0, TLS 1.1)
   - Use strong cipher suites
   - Enable HTTP Strict Transport Security (HSTS)
   - Consider implementing:
     ```powershell
     # Disable TLS 1.0 and 1.1
     New-Item 'HKLM:\SYSTEM\CurrentControlSet\Control\SecurityProviders\SCHANNEL\Protocols\TLS 1.0\Server' -Force
     New-ItemProperty -Path 'HKLM:\SYSTEM\CurrentControlSet\Control\SecurityProviders\SCHANNEL\Protocols\TLS 1.0\Server' -Name 'Enabled' -Value 0 -PropertyType 'DWord'
     
     New-Item 'HKLM:\SYSTEM\CurrentControlSet\Control\SecurityProviders\SCHANNEL\Protocols\TLS 1.1\Server' -Force
     New-ItemProperty -Path 'HKLM:\SYSTEM\CurrentControlSet\Control\SecurityProviders\SCHANNEL\Protocols\TLS 1.1\Server' -Name 'Enabled' -Value 0 -PropertyType 'DWord'
     ```

3. **Secure WinRM**: Use certificate authentication instead of basic auth for production

4. **Encrypt credentials**: Always use Ansible Vault for passwords

5. **Firewall rules**: Restrict access to known IP ranges
   ```yaml
   - name: Restrict HTTPS to specific IP
     community.windows.win_firewall_rule:
       name: "IIS HTTPS Traffic"
       localport: 443
       remoteip: "192.168.1.0/24"  # Your network
       action: allow
   ```

6. **Keep Windows updated**: Apply security patches regularly

7. **Disable unnecessary IIS features**: Remove features not in use

8. **Configure IIS security headers**:
   ```powershell
   # Add security headers
   Add-WebConfigurationProperty -PSPath 'IIS:\Sites\DefaultWebSite' -Filter "system.webServer/httpProtocol/customHeaders" -Name "." -Value @{name='X-Content-Type-Options';value='nosniff'}
   Add-WebConfigurationProperty -PSPath 'IIS:\Sites\DefaultWebSite' -Filter "system.webServer/httpProtocol/customHeaders" -Name "." -Value @{name='X-Frame-Options';value='SAMEORIGIN'}
   Add-WebConfigurationProperty -PSPath 'IIS:\Sites\DefaultWebSite' -Filter "system.webServer/httpProtocol/customHeaders" -Name "." -Value @{name='Strict-Transport-Security';value='max-age=31536000'}
   ```

9. **Certificate Management**:
   - Store certificates securely
   - Set reminders for certificate expiration
   - Use key exportable=false for production certificates
   - Regularly audit certificate store

10. **Monitor and Log**:
    - Enable IIS logging
    - Monitor certificate expiration
    - Review security logs regularly

## Files Created

- `install_iis_https.yml` - Main playbook (uses microsoft.iis collection)
- `templates/iis_index.html.j2` - HTML template for homepage
- `templates/iis_style.css.j2` - CSS stylesheet template
- `requirements.yml` - Ansible collections requirements
- `IIS_SETUP.md` - This documentation
- `VAULT_SETUP.md` - Ansible Vault setup guide
- `group_vars/windows_servers/vault.yml.example` - Example vault file

## Related Documentation

- [IIS Windows Feature Names](https://docs.microsoft.com/en-us/iis/install/installing-iis-85/installing-iis-85-on-windows-server-2012-r2)
- [Ansible Windows Modules](https://docs.ansible.com/ansible/latest/collections/ansible/windows/)
- [Community Windows Collection](https://docs.ansible.com/ansible/latest/collections/community/windows/)
- [WinRM Setup](https://docs.ansible.com/ansible/latest/user_guide/windows_setup.html)

## Example Inventory File

Create or update `hosts` file:

```ini
# Windows IIS Servers
[windows_servers]
win-iis-01 ansible_host=192.168.1.100
win-iis-02 ansible_host=192.168.1.101

[windows_servers:vars]
ansible_user=Administrator
ansible_password=P@ssw0rd123  # Use Vault in production!
ansible_connection=winrm
ansible_winrm_transport=basic
ansible_winrm_server_cert_validation=ignore
ansible_port=5985

# Custom IIS variables
iis_domain_name=www.example.com
iis_site_name=DefaultWebSite
```

## Support

For issues or questions:
- Review Ansible documentation
- Check Windows Event Viewer logs
- Verify WinRM connectivity
- Test IIS functionality manually
