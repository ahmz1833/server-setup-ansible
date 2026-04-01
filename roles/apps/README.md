# Ansible role: `ahmz1833.server_setup.apps`

Declarative **Docker container** lifecycle on the target host using **`community.docker`**: validate definitions, log in to registries, create networks, resolve **`depends_on`** order, deploy with **`docker_container`**, optionally wait on Docker **health checks**, and optionally **prune** containers or networks that are not part of **`apps_list`**.

This role is intended as an alternative to ad-hoc `docker compose` for stacks you want fully described in Ansible variables.

---

## Requirements

| Requirement | Notes |
|-------------|--------|
| **Target OS** | Tested with **Debian / Ubuntu** hosts where Docker Engine is available (`meta/main.yml` platforms). |
| **Ansible** | 2.14+ |
| **Collection** | **`community.docker`** — declared in role `meta/main.yml`; also listed in the collection `galaxy.yml` dependencies. |
| **Docker on the target** | Engine API reachable by the modules; Python **`docker`** SDK on the controller/target as required by `community.docker` (often satisfied by installing **`python3-docker`** on the host, e.g. via the **`node`** role in this collection). |

---

## Execution flow

| Step | Task file | Purpose |
|------|-----------|---------|
| 1 | `prepare.yml` | Assert schema (names, duplicates, `depends_on`), create **`{{ apps_base_path }}`** and per-app dirs, **`docker_login`** for **`apps_registries`**. |
| 2 | `network.yml` | Create **`apps_networks`**; auto-create any **bridge** networks referenced by apps but not explicitly defined. |
| 3 | `deploy.yml` | Topological sort by **`depends_on`**, inspect existing containers, **`include_tasks: container.yml`** per app. |
| 4 | `prune.yml` | If **`apps_prune.enabled`**, remove unmanaged containers and/or networks (see below). |

---

## Tags

| Tag | What runs |
|-----|-----------|
| `apps` | All of the above (each `include_tasks` is tagged). |
| `ci-cd`, `deploy` | Prepare, networks, deploy — **not** prune unless you also match prune conditions. |
| `apps-prune` | Prune task file is tagged; prune **still runs only when** **`apps_prune.enabled`** is true. |

---

## Global variables

Defaults are in `defaults/main.yml`. Commonly used:

| Variable | Default | Description |
|----------|---------|-------------|
| `apps_list` | `[]` | List of app dicts (see schema below). |
| `apps_base_path` | `/opt/apps` | Base directory; each app gets `{{ apps_base_path }}/{{ name }}/`. |
| `apps_preserve_mode` | `false` | When true, omitted keys for an **existing** container are filled from the live container (CI/CD image-only bumps). |
| `apps_default_restart_policy` | `unless-stopped` | Used when not set on the app. |
| `apps_default_pull_policy` | `missing` | Maps to `pull`: `always` / `missing` / `never` (or bool for `always`). |
| `apps_default_log_driver` / `apps_default_log_options` | `json-file`, size/rotation | Applied when not overridden per app. |
| `apps_management_labels_enabled` | `true` | Adds `apps.ansible.managed=true` and `apps.ansible.name`. |
| `apps_registries` | `[]` | `registry`, `username`, `password`, optional `reauthorize`. |
| `apps_networks` | `[]` | Explicit `docker_network` definitions (`name`, `driver`, `internal`, `ipam`, …). |

### Pruning: `apps_prune`

**Dangerous when enabled.** Removes resources **not** named in **`apps_list`** (containers) or not in the managed network set (networks).

| Key | Default | Description |
|-----|---------|-------------|
| `enabled` | `false` | Master switch. |
| `containers` | `true` | Remove unmanaged containers (after filters). |
| `networks` | `false` | Remove Docker networks not referenced by role + not `bridge`/`host`/`none`. |
| `whitelist_patterns` | includes `^sandbox-.*$` in defaults | Regex on container **name**; matches are **never** removed. **Prune logic also always prepends** `^sandbox-.*$` so clearing this list in YAML does not drop that safety net. |
| `whitelist_labels` | `[]` | Skip removal if label **key** exists (string entry) or **key/value** match (`{ key:, value: }`). |

Network pruning uses **`failed_when: false`** on removal (busy networks may fail).

---

## App schema (`apps_list` entries)

### Required

| Key | Description |
|-----|-------------|
| `name` | Container name; must match `^[a-zA-Z0-9][a-zA-Z0-9_.-]*$`. |
| `image` | Image reference — **required** for **`started`**. **Optional** for **`stopped`** and **`absent`** (module does not need an image to stop/remove). |

### Common optional keys

| Key | Description |
|-----|-------------|
| `state` | `started` (default), `stopped`, or `absent`. |
| `preserve_mode` | Override global **`apps_preserve_mode`** for this container. |
| `depends_on` | List of other **`name`** values in **`apps_list`**; those containers start (and become healthy if waited) **before** this one. Cycles are rejected. |
| `ports` | Same strings as `docker_container` (e.g. `8080:80`, `127.0.0.1:5432:5432`). |
| `networks` | e.g. `[{ name: frontend }, { name: backend, ipv4_address: 172.25.0.10 }]`. |
| `volumes` | Bind paths starting with **`./`** are resolved under **`{{ apps_base_path }}/{{ name }}/`** (only the `./` prefix is rewritten). Other forms (`named_volume:/path`, absolute binds) are passed through. |
| `env` | Dict. With **preserve mode**, merged **on top of** the running container’s env. |
| `healthcheck` / `healthcheck_wait` / `healthcheck_wait_timeout` | Docker healthcheck dict; wait loop until `healthy` or no health state (`none`), default timeout **300** s, poll every **5** s. |
| `pull`, `recreate`, `restart`, `restart_policy` | Passed through to **`docker_container`**. |
| `dir_owner`, `dir_group`, `dir_mode` | Ownership/mode for **`{{ apps_base_path }}/{{ name }}/`**. |

Many other **`docker_container`** options are mapped in `tasks/container.yml`: `command`, `entrypoint`, `user`, `working_dir`, `hostname`, `dns`, `tmpfs`, resources, capabilities, `read_only`, `privileged`, `security_opts`, `log_driver`, `log_options`, `labels`, etc.

### Idempotency (`comparisons`)

The deploy task sets:

- `image: strict`
- `env: strict`
- `labels: allow_more_present`
- `volumes: allow_more_present`
- `networks: allow_more_present`

### Preserve mode (CI/CD)

When **`apps_preserve_mode`** (or per-app **`preserve_mode`**) is **true** and the container **already exists**, omitted fields are filled from **`docker_container_info`** (ports, networks, env merge, volumes, healthcheck, resources, …). Your playbook can ship only **`name`** + **`image`** (and e.g. **`pull: always`**) while secrets and bindings stay on the host.

---

## Examples

### Minimal single container

```yaml
apps_list:
  - name: whoami
    image: traefik/whoami:v1.10
    ports:
      - "8080:80"
```

### Data directory under `apps_base_path`

Host path: **`/opt/apps/uptime-kuma/data`** → container **`/app/data`**.

```yaml
apps_base_path: /opt/apps
apps_list:
  - name: uptime-kuma
    image: louislam/uptime-kuma:1
    ports:
      - "3001:3001"
    volumes:
      - "./data:/app/data"
    restart_policy: unless-stopped
```

### Dependency order + health gating

```yaml
apps_list:
  - name: db
    image: postgres:15-alpine
    restart_policy: unless-stopped
    env:
      POSTGRES_USER: admin
      POSTGRES_PASSWORD: "{{ vault_pg_password }}"
    volumes:
      - "./pgdata:/var/lib/postgresql/data"
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U admin"]
      interval: 10s
      retries: 5

  - name: api
    image: ghcr.io/example/api:1.2.0
    depends_on:
      - db
    ports:
      - "8080:3000"
    env:
      DATABASE_URL: "postgres://admin@db:5432/app"
```

### Preserve mode (image bump only)

```yaml
apps_preserve_mode: true
apps_list:
  - name: api
    image: "ghcr.io/example/api:{{ release_tag }}"
    pull: always
```

### Stop or remove without image

```yaml
apps_list:
  - name: old_worker
    state: stopped

  - name: legacy_app
    state: absent
```

### Prune with label protection

```yaml
apps_prune:
  enabled: true
  containers: true
  networks: false
  whitelist_labels:
    - "com.docker.compose.project"
    - key: "com.example.role"
      value: "system"

apps_list:
  - name: prometheus
    image: prom/prometheus:latest
    ports:
      - "9090:9090"
```

More snippets: **`examples/apps.yml`**.

---

## Playbook usage

```yaml
- hosts: app_servers
  roles:
    - role: ahmz1833.server_setup.apps
      vars:
        apps_base_path: /opt/production
        apps_list:
          - name: web
            image: nginx:alpine
            ports:
              - "80:80"
```

---

## License

MIT
