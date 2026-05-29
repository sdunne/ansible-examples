# TLS Certificate Expiry Check

Two playbooks are available to check TLS certificate expiry:

## 1. check_cert_expiry.yml (Server-Side Check)

Checks certificates directly on the servers by examining files and certificate stores.

**Features:**
- Apache (RHEL 9): Scans `/etc/pki/tls/certs` for certificate files
- IIS (Windows): Queries IIS site bindings and certificate store via PowerShell
- Shows certificate details, expiry dates, and days remaining
- Alerts if certificates expire within 30 days (configurable)

**Usage:**
```bash
ansible-navigator run -m stdout check_cert_expiry.yml -i hosts --ask-vault-pass
```

**Requirements:**
- `community.crypto` collection (for Apache certificate parsing)
- WinRM access to Windows servers
- Proper credentials in secrets.yml
- Execution environment with necessary collections installed

## 2. check_cert_expiry_remote.yml (Client-Side Check) ⭐ Recommended

Checks certificates by connecting to HTTPS endpoints (tests what clients see).

**Features:**
- Connects to each HTTPS endpoint using OpenSSL
- Tests the actual certificate served to clients
- Works from your local machine/control node
- Displays a formatted summary report at the end

**Usage:**
```bash
ansible-navigator run -m stdout check_cert_expiry_remote.yml
```

**Requirements:**
- OpenSSL must be available in the execution environment image
- If not available, run with `--ee false` to use local environment

**Edit the endpoints in the playbook:**
```yaml
endpoints:
  - name: "Apache RHEL9"
    host: "test.sdunne.net"
    port: 443
  - name: "Windows IIS"
    host: "test1.sdunne.net"
    port: 443
```

## Configuration

Both playbooks use a configurable warning threshold:
```yaml
cert_warning_days: 30  # Alert if cert expires within 30 days
```

## Output

Both playbooks display:
- Certificate subject and issuer
- Expiration date
- Days remaining until expiry
- Status (OK, WARNING, or EXPIRED)

The remote check displays a formatted summary report at the end of the playbook run.

## Recommendations

- **Remote check**: Best for monitoring and automated checks (cron jobs) ⭐
- **Server-side check**: Best for detailed certificate inventory and troubleshooting
- Run weekly or monthly to stay ahead of expiring certificates
- Consider using the remote check in a CI/CD pipeline or monitoring system

## Troubleshooting

### OpenSSL not found in execution environment
If you get errors about openssl not being available, run without the execution environment:
```bash
ansible-navigator run -m stdout check_cert_expiry_remote.yml --ee false
```

### Connection timeouts
The remote check uses a 10-second timeout for each endpoint. If legitimate connections are timing out, increase the timeout value in the playbook's shell commands.
