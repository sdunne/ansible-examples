# acme_digicert

Gets a DigiCert DV certificate, including **wildcard + SANs**, over ACME with External Account Binding (EAB). Domain control is proven with dns-01 TXT records that the role creates and then removes through the GoDaddy API.

## CertCentral setup

1. Go to **CertCentral → Automation → ACME Directory URLs → Add ACME Directory URL**.
2. Choose a **DV product**. For a wildcard cert it must be a wildcard-capable DV product (e.g. GeoTrust DV SSL Wildcard, RapidSSL Wildcard). The product is bound to this URL.
3. Copy the **ACME directory URL**, the **Key ID (KID)**, and the **HMAC key**.

## Required variables

| Variable | Description |
|---|---|
| `acme_digicert_directory` | ACME directory URL from CertCentral |
| `acme_digicert_email` | Account contact email |
| `acme_digicert_eab_kid` / `acme_digicert_eab_hmac_key` | EAB credentials from CertCentral |
| `acme_digicert_domain` | Common name, e.g. `*.example.com` |
| `acme_digicert_godaddy_key` / `acme_digicert_godaddy_secret` | GoDaddy production API key/secret |

## Common optional variables

| Variable | Default | Notes |
|---|---|---|
| `acme_digicert_sans` | `[]` | Extra names, added to the SAN list along with the CN |
| `acme_digicert_godaddy_zones` | last two labels of CN | List every GoDaddy zone the names live in |
| `acme_digicert_cert_dir` | `./certs` | Output directory |
| `acme_digicert_cert_name` | CN with `*` → `wildcard` | Base filename |
| `acme_digicert_key_type` | `RSA` | `RSA` (`key_size`) or `ECC` (`key_curve`) |
| `acme_digicert_remaining_days` | `30` | Re-run is a no-op until within this window |
| `acme_digicert_dns_wait_seconds` | `90` | Pause after creating TXT records |

See `defaults/main.yml` for the rest.

## Wildcard notes

`*.example.com` and `example.com` validate at the same record, `_acme-challenge.example.com`, with **different** values. The role loops over `challenge_data_dns` and sends every value for a record in one GoDaddy PUT, so neither value overwrites the other. TXT records are removed in an `always:` block, so cleanup also runs when validation fails.

## Example

```yaml
- hosts: localhost
  gather_facts: false
  vars_files: [secrets.yml]
  roles:
    - role: acme_digicert
      vars:
        acme_digicert_domain: "*.example.com"
        acme_digicert_sans: [example.com, www.example.org]
        acme_digicert_godaddy_zones: [example.com, example.org]
```

Output: `certs/wildcard.example.com.{key,csr,crt}`, `-chain.crt`, and `-fullchain.crt`.

## Requirements

`community.crypto` (with `cryptography`) in the execution environment.
