# Ansible role: `ahmz.server_setup.acme`

Let’s Encrypt (ACME v2) certificate issuance and renewal with **cryptography on the Ansible controller**. The managed host only receives the final key and full chain; it does **not** need `certbot`, `acme.sh`, or similar.

---

## How it works

1. **Controller** (tasks delegated to `localhost`): create or reuse an account key, register the ACME account, generate per-certificate keys and CSRs, talk to the CA, complete challenges.
2. **Target host**: optional webroot writes or temporary standalone HTTP server for HTTP-01; receive `privkey.pem` and `fullchain.pem` via `copy`.
3. **Renewal**: read the **existing** certificate on the target with `community.crypto.x509_certificate_info`; issue only if it is missing or expires within `acme_renew_days`.
4. **After deploy**: notify **`reload certificates`** → custom command and/or `ansible.builtin.service` reload (not hard-coded to systemd only).

---

## Requirements

| Requirement | Notes |
|-------------|--------|
| **Ansible** | 2.14+ (see `meta/main.yml`). |
| **`community.crypto`** | Declared in role `meta/main.yml`; install on the controller, e.g. `ansible-galaxy collection install community.crypto`. |
| **Controller Python** | OpenSSL-backed crypto as required by `community.crypto` (typically `cryptography` + system OpenSSL). |
| **Target** | Unix-like host with `become` for writing cert paths and (for HTTP-01) webroot or port 80. **Windows targets are not supported** (bash paths, `python3 -m http.server`, etc.). |
| **Facts** | `gather_facts` should be enabled so `ansible_facts['date_time']` and handler conditions (`service_mgr`) behave correctly. |

---

## Global variables

| Variable | Description | Default |
|----------|-------------|---------|
| `acme_directory_url` | ACME directory (production or staging). | `https://acme-v02.api.letsencrypt.org/directory` |
| `acme_account_email` | Account contact (`mailto:`) for the CA. | `admin@example.com` |
| `acme_local_workspace` | Directory on the **controller** for account key, CSRs, and issued files (per CN under this path). | `/tmp/ansible_acme_workspace` |
| `acme_noninteractive` | If `true`: no `pause` prompts; DNS-01 **fails** immediately; HTTP-01 **fails** if port 80 is busy and webroot validation did not succeed. For CI / `-e` runs. | `false` |
| `acme_renew_days` | Renew if the certificate is not valid this many days ahead. | `30` |
| `acme_default_cert_dir` | Default parent for remote cert material (unless `cert_dir` / explicit paths override). | `/etc/ssl/certs` |
| `acme_webroot_path` | Webroot used for HTTP-01 when the role can place challenge files. | `/var/www/acme` |
| `acme_webroot_owner` | Owner for webroot dirs/files (optional). | `www-data` (via `default` in tasks) |
| `acme_webroot_group` | Group for webroot dirs/files (optional). | `www-data` |
| `acme_webroot_test_timeout` | Seconds for the HTTP check from the controller to `http://<CN>/.well-known/...`. | `5` |
| `acme_reload_service` | Service name passed to `ansible.builtin.service` with `state: reloaded` after deploy. Set to `""` to skip. | `nginx` |
| `acme_reload_command` | If non-empty, run this command with `become` instead of reloading `acme_reload_service`. | `""` |

Staging example:

```yaml
acme_directory_url: "https://acme-staging-v02.api.letsencrypt.org/directory"
```

---

## `acme_certificates` (list)

Each item describes one certificate.

| Key | Required | Description |
|-----|----------|-------------|
| `domains` | yes | First element = CN; rest = SANs. Wildcards require **dns-01**. |
| `challenge` | no | `dns-01` (default) or `http-01`. |
| `cert_dir` | no | Remote directory base (default: `acme_default_cert_dir/<CN>`). |
| `fullchain_path` | no | Remote full chain path (default: `<cert_dir>/fullchain.pem`). |
| `privkey_path` | no | Remote private key path (default: `<cert_dir>/privkey.pem`). |
| `renew_days` | no | Override `acme_renew_days` for this cert only. |

---

## Challenge modes

### DNS-01

- Required for **wildcard** names.
- The role **does not** create DNS records. It **pauses** and prints TXT record names/values; you create them at your DNS provider, then continue.
- With `acme_noninteractive: true`, the role **stops** with an error (use HTTP-01, interactive mode, or external DNS automation outside this role).

### HTTP-01

1. **Webroot path** (preferred): creates `.well-known/acme-challenge` under `acme_webroot_path`, probes `http://<first-domain>/.well-known/...` **from the controller**. If that returns the test token, challenges are served via the existing web server (no port 80 binding on the host by the role for validation).
2. **Standalone fallback**: temporary files under `/tmp/acme_standalone` and `python3 -m http.server` on **port 80** (needs **root** / capabilities on the target). If something already listens on 80, the role either **prompts** (interactive) or **fails** (`acme_noninteractive: true`).

**Webroot check:** The hostname in the URL is the certificate **CN** (`domains[0]`). That name must **resolve and reach** the target from the machine running the `uri` task (usually the controller).

**Port 80 “auto” mode:** The role uses `ps` **comm** names with `ansible.builtin.service`. The process name is not always the same as the init unit name; if stop/restore fails, prefer **manual** free-port 80 or fix webroot reachability.

---

## Interactive vs non-interactive

| Scenario | Suggestion |
|----------|------------|
| Laptop / shell with TTY | `acme_noninteractive: false` (default). |
| CI, `cron`, AWX without survey | `acme_noninteractive: true`, **http-01** with a **working webroot**, free port 80 if standalone is possible, or skip this role and use another ACME path. |
| DNS-01 in CI | Not supported inside this role without human or external DNS API. |

---

## Handlers

- **`reload certificates`**: runs `acme_reload_command` if set; otherwise `service: name={{ acme_reload_service }} state=reloaded` when `acme_reload_service` is non-empty.
- **`systemd daemon-reload`**: runs only when `ansible_facts.service_mgr == 'systemd'` (safe on non-systemd targets).

---

## Example: DNS-01 (wildcard)

```yaml
- hosts: webservers
  become: true
  vars:
    acme_account_email: security@example.com
    acme_certificates:
      - domains:
          - example.com
          - "*.example.com"
        challenge: dns-01
  roles:
    - role: ahmz.server_setup.acme
```

Default remote layout: `fullchain.pem` and `privkey.pem` under `/etc/ssl/certs/example.com/` (unless you set `cert_dir` / explicit paths).

---

## Example: HTTP-01 + custom paths + custom reload

```yaml
- hosts: webservers
  become: true
  vars:
    acme_account_email: admin@example.com
    acme_reload_command: /usr/local/bin/reload-app-certs.sh
    acme_certificates:
      - domains:
          - app.example.com
        challenge: http-01
        fullchain_path: /opt/myapp/ssl/cert.pem
        privkey_path: /opt/myapp/ssl/key.pem
        renew_days: 15
  roles:
    - role: ahmz.server_setup.acme
```

---

## Example: Controller == target (`localhost`)

Use a normal play with `hosts: localhost`, `connection: local`, and `become: true` so the same machine can write `/etc/...` and run webroot/standalone steps. Ensure the HTTP-01 probe can reach `http://<CN>/...` (DNS/hosts as needed).

```yaml
- hosts: localhost
  connection: local
  become: true
  vars:
    acme_noninteractive: true
    acme_reload_service: nginx
    acme_webroot_path: /var/www/acme
    acme_certificates:
      - domains: [example.com]
        challenge: http-01
  roles:
    - role: ahmz.server_setup.acme
```

---

## Limitations (by design)

- No **ACME client** on the target; no **DNS provider API** inside the role.
- **DNS-01** needs human or external automation for TXT records.
- **HTTP-01 standalone** needs **Python 3**, **`/bin/bash`**, and privileges for **port 80**.
- **Wildcard** certificates require **dns-01**.
- Empty **`acme_certificates`** runs account/workspace setup only and issues **no** certs.

---

## License

MIT
