# Let's Encrypt Certificate with ACME and GoDaddy DNS

This playbook obtains SSL/TLS certificates from Let's Encrypt using the ACME protocol with DNS-01 challenge validation via GoDaddy.

## Prerequisites

1. **Ansible Collections**: Install required collections
   ```bash
   ansible-galaxy collection install -r requirements.yml
   ```

2. **GoDaddy API Credentials**: Obtain API key and secret from [GoDaddy Developer Portal](https://developer.godaddy.com/keys)

3. **DNS Access**: Ensure your domain is managed by GoDaddy DNS

## Setup

1. **Install dependencies**:
   ```bash
   ansible-galaxy collection install -r requirements.yml
   ```

2. **Configure variables**:
   ```bash
   cp vars.example.yml vars.yml
   # Edit vars.yml with your domain and credentials
   ```

3. **Set GoDaddy credentials** (choose one method):
   
   **Option A: Environment variables** (recommended):
   ```bash
   export GODADDY_API_KEY="your_key_here"
   export GODADDY_API_SECRET="your_secret_here"
   ```
   
   **Option B: Variables file**:
   Edit `vars.yml` and encrypt with ansible-vault:
   ```bash
   ansible-vault encrypt vars.yml
   ```

## Usage

### Testing with Staging Environment (Recommended First)

The playbook uses Let's Encrypt staging by default:

```bash
ansible-playbook acme_letsencrypt.yml -e @vars.yml
```

If you encrypted vars.yml:
```bash
ansible-playbook acme_letsencrypt.yml -e @vars.yml --ask-vault-pass
```

### Production Certificates

Once tested, switch to production by editing `vars.yml`:
```yaml
acme_directory: https://acme-v02.api.letsencrypt.org/directory
```

**Warning**: Let's Encrypt has rate limits for production (5 certificates per domain per week).

## What the Playbook Does

1. Creates certificate directory
2. Generates ACME account key (if needed)
3. Generates private key for certificate
4. Creates Certificate Signing Request (CSR)
5. Initiates ACME challenge with Let's Encrypt
6. Creates DNS TXT records at GoDaddy for validation
7. Waits for DNS propagation (60 seconds)
8. Completes ACME validation
9. Downloads certificate
10. Cleans up DNS TXT records

## Output Files

Certificates are stored in `./certs/` directory:
- `account.key` - ACME account private key
- `<domain>.key` - Certificate private key
- `<domain>.csr` - Certificate signing request
- `<domain>.crt` - Certificate only
- `<domain>-fullchain.crt` - Certificate + intermediate chain
- `<domain>-chain.crt` - Intermediate chain only

## Troubleshooting

### DNS Propagation Issues
If validation fails, try increasing the wait time:
```yaml
- name: Wait for DNS propagation
  ansible.builtin.pause:
    seconds: 120  # Increase to 120 seconds
```

### Check DNS TXT Records Manually
```bash
dig +short TXT _acme-challenge.example.com @8.8.8.8
```

### GoDaddy API Issues
- Verify API credentials are correct
- Ensure API key has production access (if using production)
- Check domain is in your GoDaddy account

### Rate Limiting
Staging environment has higher rate limits. Always test there first.

## Certificate Renewal

Certificates expire after 90 days. To renew, simply run the playbook again. The playbook checks `remaining_days: 30` and will renew if less than 30 days remain.

## Security Notes

1. Protect your private keys and ACME account key
2. Use ansible-vault to encrypt sensitive variables
3. Restrict file permissions on certificate directory
4. Never commit API credentials to version control

## References

- [Let's Encrypt ACME](https://letsencrypt.org/docs/challenge-types/)
- [GoDaddy API Docs](https://developer.godaddy.com/doc)
- [Ansible community.crypto](https://docs.ansible.com/ansible/latest/collections/community/crypto/)
