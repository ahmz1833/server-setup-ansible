# Ansible role: `ahmz.server_setup.node`

Optional **host-level services**: sing-box, shell tools + bootstrap, **GOST**, **X-UI**, **Docker** (static engine + Buildx/Compose plugins), and **Prometheus node_exporter**. Binaries are deployed idempotently via the **`ahmz.server_setup.asset`** role (same collection).

Designed for **systemd** Linux. **Gather facts** must be enabled so `ansible_facts['architecture']` and OS family checks work.

---

## Requirements

| Requirement | Notes |
|-------------|--------|
| **Collection** | This role lives in `ahmz.server_setup`; install the collection and its [declared dependencies](https://github.com/ahmz1833/server-setup/blob/main/galaxy.yml) (`community.general`, `community.crypto`, `ansible.posix`, …). |
| **Role dependency** | **`asset`** (declared in `meta/main.yml`) for downloads, optional local staging, and package-mode extracts. |
| **Target** | **systemd**; privilege escalation (`become`) for installs, units, and `/usr/local` paths. |
| **Docker engine** | Static tarball from Docker is **glibc-linked**. The role **fails** the install path on **Alpine/musl** (use distro Docker or disable `node_docker_enabled`). |
| **Sing-box + `links`** | If you use `node_singbox_outbounds.links`, **`vpnparser`** must be on the **Ansible controller** (`go install github.com/ahmz1833/vpnparser@latest` and `PATH` includes `$GOPATH/bin`). |

---

## Playbook-level variables (consumed via defaults)

| Playbook var | Maps to | Purpose |
|--------------|---------|---------|
| `primary_user` | `node_primary_user` | Default non-root user for docker group, bootstrap, etc. |
| `download_locally` | `node_download_locally` | Passed to `asset` (stage on controller, push to target). |
| `is_iran` | `node_internet_restricted` | Mirrors, sing-box install toggle, bootstrap URL, … |
| `enable_ipv6` | `node_enable_ipv6` | Docker `daemon.json` IPv6 behavior. |

---

## Task order and tags

`tasks/main.yml` runs (when each section’s install/toggle allows):

1. **sing-box** — `node_singbox_install`
2. **tools** — packages + fzf/eza/zoxide/starship (+ lazydocker if Docker on) + optional bootstrap
3. **GOST** — `node_gost_install`
4. **X-UI** — `node_xui_install`
5. **Docker** — `node_docker_enabled`
6. **node_exporter** — `node_exporter_install`

| Tag | Scope |
|-----|--------|
| `node` | Full role |
| `node-singbox` | sing-box |
| `node-tools` | OS packages + CLI assets |
| `node-bootstrap` | Dotfiles/bootstrap script (subset of tools) |
| `node-gost` | GOST |
| `node-xui` | X-UI |
| `node-docker` | Docker engine + plugins + units |
| `node-exporter` | node_exporter |

---

## Install toggles and service toggles

| Install / feature | Default (see `defaults/main.yml`) | Service / runtime toggle |
|-------------------|-----------------------------------|---------------------------|
| `node_singbox_install` | `{{ node_internet_restricted }}` (often `is_iran`) | `node_singbox_enabled` |
| `node_gost_install` | `true` | `node_gost_enabled` |
| `node_xui_install` | `false` | `node_xui_enabled` |
| `node_docker_enabled` | `true` | `node_docker_enabled` (also starts/stops units) |
| `node_exporter_install` | `true` | `node_exporter_enabled` |
| `node_bootstrap_enabled` | `true` | — |
| `node_lazydocker_install` | `{{ node_docker_enabled }}` | — |

Use **`_install`** to pull in files/packages; use **`_enabled`** (where present) to control **systemd** start/enabled.

---

## Architecture and downloads

Under **`defaults/main.yml` → ARCHITECTURE**, maps translate `ansible_facts['architecture']` into upstream names for:

- **Docker static** tarball (`download.docker.com` … `x86_64`, `aarch64`, `armhf`, …)
- **Buildx** / **Compose** plugin filenames (different suffix rules)
- **Tools** (eza/zoxide/starship/lazydocker/fzf/gost) triples or Go-style suffixes

Unmapped CPU families fall back to the raw architecture string (downloads may **404**). Override the relevant `node_arch_*` or `node_docker_*` fact per host if needed.

---

## Sing-box

- **`node_singbox_libc`**: `auto` (Alpine → `-musl` tarball), `glibc`, or `musl`.
- **DNS / outbounds / TUN**: see comments in `defaults/main.yml` and **`examples/singbox.yml`**.

---

## Tools and bootstrap

- **`node_tools_packages`**: names match **Debian/Ubuntu**; replace the list on **RHEL/Alpine** if packages differ (`netcat-openbsd` → `nmap-ncat`, etc.).
- **`node_bootstrap_script_url`**: switches with `node_internet_restricted` (alternate mirror vs GitHub).

---

## Docker

- Engine from **static tgz** under `/usr/local/lib/docker`, symlinks in `/usr/local/bin`, **containerd** + **docker** **systemd** units from templates.
- **Buildx** / **Compose** as CLI plugins under `/usr/local/libexec/docker/cli-plugins` with shims in `/usr/local/bin`.
- Version gate: if installed engine **meets** `node_docker_min_version`, the heavy install block is skipped (plugins still checked via `asset`).
- **`examples/docker.yml`**: network pools, logging, metrics, Compose versions.

---

## X-UI and GOST

- **X-UI**: panel settings are applied from role variables (`node_xui_*`); see **`defaults/main.yml`** and `tasks/xui.yml`.
- **GOST**: configuration is YAML in `node_gost_config`; see **`examples/gost.yml`**.

---

## node_exporter

- Binds to **`node_exporter_listen_address`** (default localhost).
- Collectors: **`node_exporter_enabled_collectors`** / **`node_exporter_disabled_collectors`** / **`node_exporter_extra_args`**.

---

## Example playbook

```yaml
- name: Node services
  hosts: app_servers
  become: true
  vars:
    primary_user: deploy
    download_locally: false

    node_docker_enabled: true
    node_docker_bip: "172.29.0.1/16"
    node_docker_default_address_pools:
      - base: "172.30.0.0/16"
        size: 24

    # Sing-box Tunneling
    node_singbox_install: true
    node_singbox_enabled: true
    node_singbox_mixed_listen: "127.0.0.1"
    node_singbox_mixed_port: 10808
    node_singbox_outbounds:
      links:
        - "vless://uuid@server:443?security=tls#my-proxy"
      selectors:
        - tag: auto
          type: urltest
          use_all_links: true

    # Monitoring
    node_exporter_install: true
    node_exporter_enabled: true
    node_exporter_enabled_collectors:
      - systemd

    node_gost_install: true
    node_gost_enabled: false
    node_gost_config: {}

  roles:
    - role: ahmz.server_setup.node
```

More patterns: **`roles/node/examples/`** (`docker.yml`, `singbox.yml`, `gost.yml`).

---

## Handlers

Reload/restart **docker**, **containerd**, **sing-box**, **gost**, **x-ui**, **node_exporter** when templates or installs notify them; includes **systemd daemon-reload** where needed.

---

## License

MIT
