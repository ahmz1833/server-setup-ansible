# Ansible Role: ahmz.server_setup.acme

A highly secure, controller-centric Let's Encrypt / ACME certificate issuance and renewal engine.

## Description

The `acme` role automates the generation and renewal of SSL/TLS certificates. Unlike traditional setups that install `certbot` on the target server, this role performs all cryptographic operations (Account Registration, Key Generation, CSR Generation, ACME API handshakes) **locally on the Ansible control node**. It then securely pushes the finalized certificates to the remote server.

### Core Capabilities
* **Zero Remote Dependencies:** Does not require `certbot`, `acme.sh`, or any ACME client on the target servers.
* **Interactive DNS-01 Challenges:** Automatically pauses the playbook, displays the required `_acme-challenge` TXT records, and waits for your confirmation to proceed.
* **Smart HTTP-01 Challenges:** 1. Detects if an existing webroot is accessible from the internet. If so, it uses it with zero downtime.
  2. If port 80 is blocked/used by another service, it prompts to safely suspend the service, spins up a native Python 3 web server to handle the ACME challenge, and restores the original service afterward.
* **Mathematical Idempotency:** Parses the remote X.509 certificate to check its exact expiration date. Only triggers a renewal if the certificate expires within the configured threshold (default: 30 days).

---

## ⚠️ Important Execution Requirement

Because this role utilizes `ansible.builtin.pause` to prompt for DNS record creation or Port 80 conflict resolution, **it must be run interactively in a TTY/terminal by a human**. It is currently not designed for unattended CI/CD pipelines unless the HTTP-01 webroot path is perfectly configured and accessible beforehand.

---

## Role Variables: Global Configuration

| Variable | Description | Default |
|---|---|---|
| `acme_directory_url` | The ACME API endpoint. | `https://acme-v02.api.letsencrypt.org/directory` |
| `acme_account_email` | Email used for Let's Encrypt registration/notifications. | `admin@example.com` |
| `acme_local_workspace` | Directory on the **Ansible Controller** to store keys. | `/tmp/ansible_acme_workspace` |
| `acme_renew_days` | Number of days before expiration to trigger a renewal. | `30` |
| `acme_default_cert_dir`| Default remote directory to push deployed certificates to. | `/etc/ssl/certs` |
| `acme_webroot_path` | Target webroot for HTTP-01 zero-downtime challenges. | `/var/www/acme` |
| `acme_reload_service` | systemd service to reload after a successful deployment. | `nginx` |
| `acme_reload_command` | Custom shell command to run on reload (overrides service). | `""` |

## Role Variables: Certificates (`acme_certificates`)

The `acme_certificates` variable is a list of dictionaries. Each dictionary represents a unique certificate request.

| Key | Type | Description |
|---|---|---|
| `domains` | List | **(Required)** List of domains. The first item becomes the Common Name (CN), the rest are Subject Alternative Names (SANs). |
| `challenge` | String | Challenge type: `dns-01` or `http-01`. (Default: `dns-01`) |
| `cert_dir` | String | Base directory for the certificate on the remote host. (Default: `{{ acme_default_cert_dir }}/{{ CN }}`) |
| `fullchain_path` | String | Exact remote path for `fullchain.pem`. (Default: `{{ cert_dir }}/fullchain.pem`) |
| `privkey_path` | String | Exact remote path for `privkey.pem`. (Default: `{{ cert_dir }}/privkey.pem`) |
| `renew_days` | Integer | Override the global `acme_renew_days` for this specific certificate. |

---

## Example Playbooks

### 1. Standard DNS-01 Challenge (Wildcard Support)
`dns-01` is required if you want to issue wildcard certificates (`*.example.com`). When run, Ansible will pause and display the required TXT records.

```yaml
- name: Issue Certificates
  hosts: webservers
  vars:
    acme_account_email: "security@mycompany.com"
    acme_certificates:
      - domains:
          - "example.com"
          - "*.example.com"
        challenge: "dns-01"
  roles:
    - ahmz.server_setup.acme
```
*Resulting files on target:*
* `/etc/ssl/certs/example.com/fullchain.pem`
* `/etc/ssl/certs/example.com/privkey.pem`

### 2. HTTP-01 Challenge with Custom Paths & Custom Reload
Uses `http-01` (no wildcards allowed). Instead of reloading `nginx`, it runs a custom bash script after the keys are pushed.

```yaml
- name: Issue Certificates
  hosts: webservers
  vars:
    acme_account_email: "admin@mycompany.com"
    acme_reload_command: "/usr/local/bin/restart-custom-app.sh"
    acme_certificates:
      - domains:
          - "app.example.com"
        challenge: "http-01"
        fullchain_path: "/opt/myapp/ssl/cert.pem"
        privkey_path: "/opt/myapp/ssl/key.pem"
        renew_days: 15
  roles:
    - ahmz.server_setup.acme
```

## Handlers Integration
If you rely on the default `acme_reload_service: "nginx"`, this role will automatically trigger a systemd reload whenever a certificate is newly issued or renewed, ensuring the web server picks up the fresh certificates immediately without dropping active connections.

## Dependencies
* `community.crypto` collection must be installed on the Ansible controller.
* `cryptography` Python library must be installed on the Ansible controller.

## License
MIT
