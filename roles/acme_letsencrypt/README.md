# Ansible Role: acme_letsencrypt

Obtain Let's Encrypt SSL/TLS certificates using the ACME protocol with GoDaddy DNS-01 challenge validation.

## Description

This role automates the process of obtaining SSL/TLS certificates from Let's Encrypt using DNS-01 validation via the GoDaddy API. It handles the complete certificate lifecycle:
- Generates RSA private keys
- Creates Certificate Signing Requests (CSR)
- Registers with Let's Encrypt ACME service
- Creates temporary DNS TXT records for domain validation
- Completes the ACME challenge
- Cleans up DNS records after validation

## Requirements

### Ansible Collections

This role requires the following Ansible collections:
- `community.crypto` - For certificate management
- `community.general` - For general utilities

Install them with:
```bash
ansible-galaxy collection install community.crypto community.general
```

### GoDaddy API Credentials

You need API credentials from GoDaddy:
1. Log in to your GoDaddy account
2. Go to https://developer.godaddy.com/keys
3. Create a new API key (Production or Test/Development)
4. Note your API Key and API Secret

## Role Variables

### Required Variables

| Variable | Description | Example |
|----------|-------------|---------|
| `domain_name` | Primary domain for the certificate | `example.com` |
| `acme_email` | Email for Let's Encrypt account registration | `admin@example.com` |
| `godaddy_api_key` | GoDaddy API key | `abc123...` |
| `godaddy_api_secret` | GoDaddy API secret | `xyz789...` |

### Optional Variables

| Variable | Default | Description |
|----------|---------|-------------|
| `acme_directory` | `https://acme-staging-v02.api.letsencrypt.org/directory` | ACME endpoint (staging or production) |
| `additional_domains` | `[]` | Additional domains for SAN certificate |
| `cert_dir` | `./certs` | Directory to store certificates |
| `dns_wait_seconds` | `90` | Seconds to wait for DNS propagation |
| `account_key_path` | `{{ cert_dir }}/account.key` | Path for ACME account key |
| `cert_key_path` | `{{ cert_dir }}/{{ domain_name }}.key` | Path for certificate private key |
| `cert_path` | `{{ cert_dir }}/{{ domain_name }}.crt` | Path for certificate |
| `cert_fullchain_path` | `{{ cert_dir }}/{{ domain_name }}-fullchain.crt` | Path for full chain certificate |

### ACME Directories

**Staging (default):** `https://acme-staging-v02.api.letsencrypt.org/directory`
- Use for testing
- Certificates are not trusted by browsers
- Higher rate limits

**Production:** `https://acme-v02.api.letsencrypt.org/directory`
- Use for production certificates
- Certificates are trusted by all major browsers
- Lower rate limits (50 certificates per domain per week)

## Dependencies

None

## Example Playbook

### Basic Usage

```yaml
---
- name: Obtain Let's Encrypt certificate
  hosts: localhost
  gather_facts: false
  
  roles:
    - role: acme_letsencrypt
      vars:
        domain_name: example.com
        acme_email: admin@example.com
        godaddy_api_key: "{{ lookup('env', 'GODADDY_API_KEY') }}"
        godaddy_api_secret: "{{ lookup('env', 'GODADDY_API_SECRET') }}"
```

### Production Certificate

```yaml
---
- name: Obtain production Let's Encrypt certificate
  hosts: localhost
  gather_facts: false
  
  roles:
    - role: acme_letsencrypt
      vars:
        acme_directory: https://acme-v02.api.letsencrypt.org/directory
        domain_name: example.com
        acme_email: admin@example.com
        godaddy_api_key: "{{ lookup('env', 'GODADDY_API_KEY') }}"
        godaddy_api_secret: "{{ lookup('env', 'GODADDY_API_SECRET') }}"
```

### Multi-Domain (SAN) Certificate

```yaml
---
- name: Obtain certificate for multiple domains
  hosts: localhost
  gather_facts: false
  
  roles:
    - role: acme_letsencrypt
      vars:
        domain_name: example.com
        additional_domains:
          - www.example.com
          - api.example.com
        acme_email: admin@example.com
        godaddy_api_key: "{{ lookup('env', 'GODADDY_API_KEY') }}"
        godaddy_api_secret: "{{ lookup('env', 'GODADDY_API_SECRET') }}"
```

### Using Ansible Vault for Secrets

Create a vault file:
```bash
ansible-vault create secrets.yml
```

Add your credentials:
```yaml
---
godaddy_api_key: your_actual_api_key
godaddy_api_secret: your_actual_api_secret
```

Reference in playbook:
```yaml
---
- name: Obtain Let's Encrypt certificate
  hosts: localhost
  gather_facts: false
  
  vars_files:
    - secrets.yml
  
  roles:
    - role: acme_letsencrypt
      vars:
        domain_name: example.com
        acme_email: admin@example.com
```

Run with:
```bash
ansible-playbook playbook.yml --ask-vault-pass
```

## Usage Tips

1. **Test first:** Always test with the staging ACME directory before using production
2. **DNS propagation:** The default 90-second wait works for most cases, but you may need to increase it depending on your DNS provider
3. **Certificate renewal:** Run the role again before certificate expiration (Let's Encrypt certificates are valid for 90 days)
4. **Rate limits:** Be aware of Let's Encrypt rate limits in production

## Certificate Files Generated

After successful execution, the following files are created in `cert_dir`:

| File | Description |
|------|-------------|
| `account.key` | ACME account private key (reused for renewals) |
| `{{ domain_name }}.key` | Certificate private key |
| `{{ domain_name }}.csr` | Certificate signing request |
| `{{ domain_name }}.crt` | Certificate only |
| `{{ domain_name }}-fullchain.crt` | Certificate + intermediate chain (use this for web servers) |
| `{{ domain_name }}-chain.crt` | Intermediate certificate chain |

## License

MIT

## Author Information

Created by Sebastian Dunne
