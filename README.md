# Ansible collection: `ahmz1833.server_setup`

Roles for **Debian/Ubuntu**-style hosts: base hardening, edge Nginx (optional ModSecurity), TLS automation, Docker workloads, and optional SSH-into-container sandboxes. Binaries and static bundles are installed through a shared **`asset`** role used internally by **`node`**, **`nginx`**, and **`acme`**.

**Galaxy namespace:** [`ahmz1833`](https://galaxy.ansible.com/ui/namespaces/ahmz1833/) · **Collection name:** `server_setup` · **FQCN prefix:** `ahmz1833.server_setup.<role>`

---

## Requirements

| Requirement | Notes |
|-------------|--------|
| **Ansible** | 2.14+ (`ansible-core` compatible) |
| **Python**  | 3.9+ on the controller |
| **Targets** | Roles are written for **systemd**-based Linux; several roles assume **Debian/Ubuntu** packages or paths—see each role README. |

### Collection dependencies

Declared in `galaxy.yml`; install with the collection or via `ansible-galaxy collection install -r requirements.yml`:

| Collection | Purpose (examples) |
|------------|---------------------|
| `community.general` | `archive`, and other helpers |
| `community.crypto` | TLS / crypto (e.g. ACME) |
| `community.docker` | `docker_container`, `docker_network`, … |
| `ansible.posix` | `sysctl`, `authorized_key`, … |
| `ansible.utils` | IP / text utilities where used |

---

## Installation

### From Ansible Galaxy (after publish)

```bash
ansible-galaxy collection install ahmz1833.server_setup
```

Pin a version:

```bash
ansible-galaxy collection install ahmz1833.server_setup:==1.0.0
```

### From Git

`requirements.yml`:

```yaml
---
collections:
  - name: https://github.com/ahmz1833/server-setup.git
    type: git
    version: master
```

```bash
ansible-galaxy collection install -r requirements.yml
```

Use your repository’s default branch (`main`, `master`, or a tag) for `version`.

### Build and install from a checkout

```bash
cd /path/to/server-setup
ansible-galaxy collection build
ansible-galaxy collection install ahmz1833-server_setup-1.0.0.tar.gz
```

The artifact name is **`{namespace}-{name}-{version}.tar.gz`**.

### Publishing to Galaxy (maintainers)

1. Bump **`version`** in `galaxy.yml` (semantic versioning).
2. `ansible-galaxy collection build`
3. `ansible-galaxy collection publish ahmz1833-server_setup-<version>.tar.gz --token <GALAXY_API_TOKEN>`

Ensure the **`namespace`** in `galaxy.yml` matches your Galaxy namespace (**`ahmz1833`**).

---

## Roles overview

| Role | Purpose |
|------|---------|
| **`ahmz1833.server_setup.core`** | Timezone, APT mirrors, packages, users, SSH hardening, iptables/nftables-style firewall, sysctl, fail2ban. |
| **`ahmz1833.server_setup.node`** | Sing-box, GOST, X-UI, Docker (static engine + plugins), shell tools, `node_exporter`; uses **`asset`** for downloads. |
| **`ahmz1833.server_setup.nginx`** | Debian Nginx, ModSecurity CRS, vhosts from `nginx_sites`, exporter/Promtail optional; depends on **`asset`**. |
| **`ahmz1833.server_setup.acme`** | DNS-01 / HTTP-01 certificates; uses **`asset`** where applicable. |
| **`ahmz1833.server_setup.apps`** | Declarative `docker_container` stacks: deps, health wait, preserve mode, optional prune. |
| **`ahmz1833.server_setup.sandbox`** | SSH on a dedicated port into per-user Docker sandboxes (`blockinfile` on `sshd_config`). |
| **`ahmz1833.server_setup.asset`** | Generic download / extract / install helper (binaries, files, packages); dependency of other roles. |

Each role has its own **`roles/<name>/README.md`** for variables and examples.

---

## Quick examples

### Core

```yaml
- hosts: all
  become: true
  roles:
    - role: ahmz1833.server_setup.core
      vars:
        is_iran: false
        core_manage_users: true
        core_manage_ssh: true
        core_manage_firewall: true
```

### Node

```yaml
- hosts: all
  become: true
  roles:
    - role: ahmz1833.server_setup.node
      vars:
        node_docker_enabled: true
        node_gost_enabled: true
        node_exporter_enabled: true
```

### Nginx

```yaml
- hosts: edge
  become: true
  roles:
    - role: ahmz1833.server_setup.nginx
      vars:
        nginx_managed: true
        nginx_sites:
          - domain: example.com
            ssl_enabled: true
            upstream: "http://127.0.0.1:8080"
```

### ACME

```yaml
- hosts: all
  become: true
  roles:
    - role: ahmz1833.server_setup.acme
      vars:
        acme_account_email: admin@example.com
        acme_certificates:
          - domains:
              - example.com
              - "*.example.com"
```

### Apps (`apps_list`)

```yaml
- hosts: app_servers
  become: true
  roles:
    - role: ahmz1833.server_setup.apps
      vars:
        apps_list:
          - name: web
            image: nginx:alpine
            ports:
              - "8080:80"
```

### Sandbox

```yaml
- hosts: all
  become: true
  roles:
    - role: ahmz1833.server_setup.sandbox
      vars:
        sandbox_ssh_port: 2222
        sandbox_users:
          - name: guest1
            ssh_keys:
              - "ssh-ed25519 AAAA... your-key"
            image: ubuntu:24.04
```

Connect as **`sandbox@host`** (or your `sandbox_shared_user`) on **`sandbox_ssh_port`**; see the sandbox role README.

---

## Cross-role playbook variables

Set at play, group, or host level when you want shared behavior:

| Variable | Typical use |
|----------|-------------|
| `is_iran` | **`core`**: timezone/mirrors; **`node`**: `node_internet_restricted`; **`nginx`**: `nginx_download_locally` (via default expression). |
| `enable_ipv6` | **`core`**: sysctl/firewall IPv6; **`nginx`**: `listen [::]:…` when enabled. |
| `primary_user` | **`core`**: `core_primary_user` (protected user, SSH keys). |
| `download_locally` | **`node`**: fetch artifacts on the controller / cache (`node_download_locally`); also referenced by roles that pass it into **`asset`**. |

Exact wiring is in each role’s `defaults/main.yml`.

---

## Repository layout

- **`galaxy.yml`** — collection metadata for `ansible-galaxy collection build` / publish.
- **`roles/`** — one directory per role (`core`, `node`, `nginx`, `acme`, `apps`, `sandbox`, `asset`).
- **`playbooks/`** — optional sample playbooks (not required to use the collection).

---

## License

MIT
