# Ansible Role: ahmz.server_setup.apps

A production-ready Docker container lifecycle manager and orchestrator.

## Description

The `apps` role is a complete replacement for `docker-compose`. It manages complex, multi-container deployments directly through native Ansible Docker modules. It introduces advanced CI/CD capabilities like **Preserve Mode** (for partial updates), strict state enforcement (Terraform-style pruning), and automated dependency graph resolution.

### Core Capabilities

* **Dependency Resolution:** Automatically calculates the correct startup order based on `depends_on`.

* **Health Gating:** Pauses the deployment pipeline until a container's Docker `healthcheck` reports as `healthy` before starting its dependents.

* **Smart Preserve Mode:** Fetches the running container's state and merges it with your playbook. This allows CI/CD pipelines to update *just* the Docker image tag without needing to know the container's secrets, ports, or volumes.

* **Strict Pruning:** Can automatically destroy any containers or networks running on the host that are *not* defined in your Ansible inventory, enforcing pure Infrastructure-as-Code (with regex/label whitelists for safety).

* **Auto-Provisioning:** Automatically creates referenced Docker networks and host-mounted volume directories under `apps_base_path`.

## Available Tags

* `apps`: Runs the entire role.

* `ci-cd`: Validates, authenticates, provisions networks, and deploys containers.

* `deploy`: Alias for `ci-cd`.

* `apps-prune`: Runs the pruning task to clean up unmanaged resources.

## Role Variables: Global Configuration

| Variable | Description | Default | 
 | ----- | ----- | ----- | 
| `apps_base_path` | Host directory where app data volumes are automatically created. | `/opt/apps` | 
| `apps_preserve_mode` | Retain omitted fields (ports, envs) from running containers. | `false` | 
| `apps_management_labels_enabled` | Adds `apps.ansible.managed=true` to all created containers. | `true` | 
| `apps_registries` | List of dicts (`registry`, `username`, `password`) for private repos. | `[]` | 
| `apps_networks` | List of explicitly defined Docker networks (auto-created if omitted). | `[]` | 

### Pruning Configuration (`apps_prune`)

WARNING: Enable with caution. Removes containers not defined in `apps_list`.

```yaml
apps_prune:
  enabled: false
  containers: true
  networks: false
  whitelist_patterns: 
    - "^sandbox-.*$"
  whitelist_labels:
    - "com.docker.compose.project" # Keep docker-compose containers
```

## Role Variables: Application Schema (`apps_list`)

The `apps_list` is a list of dictionaries. Each dictionary represents a container. Almost all parameters map directly to `docker_container` arguments.

### Key Application Properties

| Key | Type | Description | 
 | ----- | ----- | ----- | 
| `name` | String | **(Required)** Container name. | 
| `image` | String | **(Required)** Docker image and tag. | 
| `state` | String | `started`, `stopped`, or `absent`. (Default: `started`) | 
| `preserve_mode` | Bool | Overrides global `apps_preserve_mode` for this specific container. | 
| `depends_on` | List | Names of other apps in `apps_list` that must start/be healthy first. | 
| `ports` | List | Port bindings (e.g., `["8080:80", "127.0.0.1:5432:5432"]`). | 
| `volumes` | List | Volume bindings. Relative paths (`./data:/app`) map to `apps_base_path/name/data`. | 
| `env` | Dict | Environment variables. | 
| `healthcheck` | Dict | Keys: `test`, `interval`, `timeout`, `retries`. | 
| `healthcheck_wait` | Bool | Pause deployment until container is `healthy`. (Default: `true`) | 
| `networks` | List | Networks to attach to (e.g., `[{name: frontend}, {name: backend}]`). | 

*(Supports standard Docker features: `user`, `working_dir`, `command`, `memory`, `cpus`, `privileged`, `labels`, `restart_policy`, etc.)*

## Example Playbooks

### 1. Multi-Tier Application with Health Gating

This example ensures Postgres starts, creates the host directories, and waits until Postgres is fully healthy before starting the Node.js API.

```yaml
- name: Deploy Production Stack
  hosts: app_servers
  vars:
    apps_base_path: "/opt/production"
    apps_list:
      - name: db
        image: postgres:15-alpine
        restart_policy: unless-stopped
        env:
          POSTGRES_USER: admin
          POSTGRES_PASSWORD: secretpassword
        volumes:
          - "./pgdata:/var/lib/postgresql/data" # Maps to /opt/production/db/pgdata
        healthcheck:
          test: ["CMD-SHELL", "pg_isready -U admin"]
          interval: 10s
          retries: 5
        
      - name: api
        image: myrepo/api:v1.2.0
        depends_on: [db]
        ports:
          - "8080:3000"
        env:
          DB_HOST: db
          DB_USER: admin
  roles:
    - ahmz.server_setup.apps
```

### 2. CI/CD Pipeline (Preserve Mode)

If you are running Ansible from a CI/CD pipeline (like GitLab CI or GitHub Actions), you often only want to update the image tag without hardcoding the database passwords or port bindings into the CI repository.

By using `preserve_mode: true`, Ansible fetches the running container's configuration and merges it with your playbook update.

```yaml
- name: Update API Image
  hosts: app_servers
  vars:
    # Retain all existing env vars, ports, and volumes
    apps_preserve_mode: true 
    apps_list:
      - name: api
        image: "myrepo/api:{{ target_version }}"
  roles:
    - ahmz.server_setup.apps
```

### 3. Strict Infrastructure Enforcement

Automatically destroys any container that is manually spun up (`docker run ...`) outside of this playbook, protecting against configuration drift.

```yaml
- name: Enforce State
  hosts: all
  vars:
    apps_prune:
      enabled: true
      containers: true
      whitelist_patterns:
        - "^temp-.*$" # Protects containers starting with 'temp-'
    apps_list:
      - name: monitoring
        image: prom/prometheus:latest
  roles:
    - ahmz.server_setup.apps
```

## Dependencies

* `community.docker` collection must be installed.
* Docker Engine and Python `docker` package must be present on target host (handled by the `node` role in this collection).

## License

MIT
