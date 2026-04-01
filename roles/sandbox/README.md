# Ansible role: `ahmz1833.server_setup.sandbox`

SSH gateway into **per-guest Docker containers**: one locked **host** Linux user, **forced commands** in `authorized_keys`, a **dedicated `sshd` port**, and **no** port/X11/agent forwarding. Guests never receive a host shell; they land in `sandbox-<name>` with UID/GID aligned to the host sandbox user.

---

## Requirements

| Requirement | Notes |
|-------------|--------|
| **Ansible** | 2.14+ (`meta/main.yml`). |
| **Collection** | **`community.docker`** — declared in role `meta/main.yml` (also a dependency of the collection `galaxy.yml`). |
| **Docker** | Engine for **`docker_container`**; **`docker`** CLI on the target for **`docker cp`** / **`docker exec`** during in-container bootstrap. |
| **OpenSSH** | The managed snippet uses **`Match LocalPort`** — **OpenSSH 8.9+** (e.g. Debian **bookworm+**, Ubuntu **22.04+**). On older **sshd**, replace or adjust the block manually. |
| **SSH layout** | Sandbox settings are written with **`ansible.builtin.blockinfile`** into **`/etc/ssh/sshd_config`** between markers **`# BEGIN ANSIBLE MANAGED SANDBOX SSH`** / **`# END ...`**, then checked with **`sshd -t`** (via the module’s `validate` hook). |
| **Firewall** | Allow **`sandbox_ssh_port`** (e.g. via **`core_firewall_rules`** in the **`core`** role). |

---

## What runs (order)

1. **Validate** (`tasks/validate.yml`) — fails fast if **`sandbox_users`** is empty, a row is invalid, or names duplicate (avoids prune wiping every **`sandbox-*`** by mistake).
2. **Prune** — Any Docker container whose name matches **`^sandbox-`** but is **not** in the expected set from **`sandbox_users`** is **stopped** (default) or **removed** (`sandbox_prune: true`).
3. **Provision** — Host user **`sandbox_shared_user`**, **`/usr/local/bin/sandbox-login.sh`**, **`sudoers`**, **`authorized_keys`**, **`docker_container`** per entry, **`init-sandbox.sh`** inside the container, then **`blockinfile`** for **`sshd`**.

---

## Ansible tags

| Tag | Scope |
|-----|--------|
| `sandbox` | Validation, prune, and provision. |
| `sandbox-prune` | Validation + prune only (still requires **`sandbox_managed: true`**). |

---

## Role variables

### Global (`defaults/main.yml`)

| Variable | Default | Description |
|----------|---------|-------------|
| `sandbox_managed` | `true` | When false, the role does nothing (no validation/prune/provision). |
| `sandbox_ssh_port` | `2222` | Extra **`sshd`** listener; **`Match LocalPort`** ties **`AllowUsers`** to this port. |
| `sandbox_prune` | `false` | If **`true`**, unmanaged **`sandbox-*`** containers are **removed**; if **`false`**, they are **stopped**. |
| `sandbox_shared_user` | `sandbox` | Locked host account used for SSH; clients authenticate **as this user** (see below). |

### Virtual users: `sandbox_users`

List of dictionaries. **Required per entry:** **`name`**, **`ssh_keys`** (non-empty list of full public key lines).

| Key | Description |
|-----|-------------|
| `name` | Logical guest id; container **`sandbox-<name>`**; must match `^[a-zA-Z0-9][a-zA-Z0-9_.-]*$`. |
| `image` | Image for **`docker_container`**; default **`ubuntu:24.04`** if omitted. |
| `shell` | Interactive shell inside the container (default **`/bin/bash`** in **`authorized_keys`**). |
| `ssh_keys` | Public keys allowed to map to this guest (forced `command=` per key). |
| `ports` | Published ports (e.g. **`127.0.0.1:8081:8080`**). |
| `volumes` | Bind/volume list; if omitted, **`/home/<shared_user>/<name>:/home/<name>`**. |
| `working_dir` | Container working directory (default **`/home/<name>`**). |
| `env` | Extra environment variables (merged with **`HOME=/home/<name>`**). |
| `networks` | **`docker_container`** networks list. |
| `memory` / `cpus` | Resource limits passed to **`community.docker.docker_container`**. |

---

## How clients connect

Linux login is always **`sandbox_shared_user`** on **`sandbox_ssh_port`**. The **SSH public key** selects the virtual guest via **`authorized_keys`** `command="sudo ... sandbox-login.sh <name> <shell>"`.

```bash
ssh -p 2222 -i ~/.ssh/developer1_ed25519 sandbox@server.example.com
```

Use **`-i`** when multiple keys are loaded so the correct key matches the intended **`sandbox_users`** entry.

---

## Example playbook

```yaml
- name: Deploy sandbox environments
  hosts: servers
  vars:
    core_firewall_rules:
      - { port: 2222, comment: "Sandbox SSH" }

    sandbox_managed: true
    sandbox_ssh_port: 2222
    sandbox_prune: true

    sandbox_users:
      - name: developer1
        image: ghcr.io/example/sandbox-base:latest
        shell: /bin/zsh
        ssh_keys:
          - "ssh-ed25519 AAAA... comment"
        volumes:
          - /opt/sandbox_data/dev1:/workspace
        ports:
          - "127.0.0.1:8081:8080"
        memory: 512m
        cpus: 0.5

    nginx_sites:
      - domain: dev1.example.com
        upstream: "http://127.0.0.1:8081"

  roles:
    - role: ahmz1833.server_setup.core
    - role: ahmz1833.server_setup.sandbox
    - role: ahmz1833.server_setup.nginx
```

---

## Security and limitations

- **Forwarding** is disabled in **`sshd`** and in **`authorized_keys`** options for the sandbox user.
- **`sandbox-login.sh`** runs **`docker exec`**; **`SSH_ORIGINAL_COMMAND`** is passed through **`bash -c`** for non-interactive use — avoid untrusted control of that string.
- **Prune** affects **every** container whose name matches **`^sandbox-`**, not only those created by this role; use naming discipline or adjust the role if you share that prefix.
- **`Match LocalPort`** requires a recent **OpenSSH**; verify with **`sshd -V`** before relying on this role.

---

## Handlers

- **`Restart sshd`** — **`service`** name **`ssh`** on Debian family, **`sshd`** elsewhere; tolerates missing **`ansible_facts`** by defaulting to **`sshd`**.

---

## License

MIT
