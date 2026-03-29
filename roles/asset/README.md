# Ansible Role: ahmz.server_setup.asset

A highly advanced, universal utility role for fetching, caching, extracting, and installing remote assets idempotently. 

## Description

The `asset` role acts as the primary deployment engine for binaries and packages in this collection. Instead of relying on `apt`/`dnf` for rapidly updating tools (e.g., Docker-compose, Sing-box, X-UI), this role downloads them directly from upstream release pages.

### Key Capabilities
* **Idempotency**: Runs CLI version commands against existing binaries and compares the output to the target version via Regex.
* **Two-Stage Caching (Air-gap friendly)**: Can download assets to the local Ansible control node first, cache them, compress them, and push them to the target server.
* **Proxy Support**: Native support for downloading assets via HTTP/HTTPS/SOCKS5 proxies.
* **Format Agnostic**: Handles raw binaries, single files, or extracted directories (`tar.gz`, `zip`).

## Role Variables

### Required Variables
| Variable | Description | Example |
|---|---|---|
| `asset_name` | Name of the tool. | `lazydocker` |
| `asset_type` | Type of payload. Options: `binary`, `package`, `file`. | `binary` (Default) |
| `asset_version` | Target version. | `1.0.0` or `>=2.1.0` |
| `asset_url` | Download URL template (use `%v` for version injection). | `github.com/org/app/v%v/app.tar.gz` |

### Optional / Advanced Variables
| Variable | Description | Default |
|---|---|---|
| `asset_dest` | Final absolute path. | `/usr/local/bin/{{ asset_name }}` |
| `asset_extract` | Whether to unarchive the payload. | `true` |
| `asset_extract_src` | Specific file/folder inside the archive to extract. | `{{ asset_name }}` |
| `asset_version_cmd` | Command to check existing version (Required for binaries). | `--version` |
| `asset_version_regex` | Regex to extract version string from command output. | `([0-9.]+)` |
| `asset_download_locally`| Download to control node first, then push to target. | `false` |
| `asset_service_stop` | Systemd service to stop before overwriting. | `""` |
| `asset_proxy` | Proxy URL for downloading (HTTP/SOCKS5). | `""` |
| `asset_use_mirror` | Swap origin domain with a mirror base URL. | `false` |
| `asset_register_path_var`| Variable name to dynamically register the final path. | `""` |

## Available Tags

* `utility`
* `installer`
* `downloader`

## Example Playbooks

### 1. Installing a raw binary (Docker Compose)
```yaml
- name: Deploy Docker Compose
  hosts: servers
  tasks:
    - name: Install Docker Compose via Asset Role
      ansible.builtin.include_role:
        name: ahmz.server_setup.asset
      vars:
        asset_name: "docker-compose"
        asset_version: "2.24.5"
        asset_url: "[github.com/docker/compose/releases/download/v%v/docker-compose-linux-x86_64](https://github.com/docker/compose/releases/download/v%v/docker-compose-linux-x86_64)"
        asset_extract: false
        asset_version_cmd: "version --short"
        asset_version_regex: 'v?([0-9]+\.[0-9]+\.[0-9]+)'
```

### 2. Installing an extracted binary with local caching
Useful for servers with restricted internet access. Downloads to the control node first.
```yaml
- name: Deploy Sing-box
  hosts: restricted_servers
  tasks:
    - name: Install Sing-box using local proxy bridge
      ansible.builtin.include_role:
        name: ahmz.server_setup.asset
      vars:
        asset_name: "sing-box"
        asset_version: "1.8.0"
        asset_url: "[github.com/SagerNet/sing-box/releases/download/v%v/sing-box-%v-linux-amd64.tar.gz](https://github.com/SagerNet/sing-box/releases/download/v%v/sing-box-%v-linux-amd64.tar.gz)"
        asset_extract: true
        asset_extract_src: "sing-box-%v-linux-amd64/sing-box"
        asset_version_cmd: "version"
        asset_version_regex: 'sing-box version ([0-9.]+)'
        asset_service_stop: "sing-box"
        asset_download_locally: true
        asset_proxy: "socks5://127.0.0.1:1080"
```

## License
MIT
