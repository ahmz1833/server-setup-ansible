# Ansible role: `ahmz1833.server_setup.asset`

Idempotent **download, extract, and install** helper for upstream release artifacts. Used across this collection (`node`, `nginx`, `acme`, …) instead of pinning every tool in OS package managers.

---

## Requirements

| Requirement | Notes |
|-------------|--------|
| **Ansible** | 2.14+ (`meta/main.yml`). |
| **Collections** | **`community.general`** (`archive` module for gzip payload when `asset_download_locally` is true). Declared in role `meta/main.yml`. |
| **Targets** | Unarchive / copy need standard tools (`tar`, `zip` support per `unarchive`). **systemd** is assumed only when **`asset_service_stop`** is set (`ansible.builtin.systemd`). |
| **Controller** | For **`asset_download_locally: true`**, cache and staging run on **localhost**; ensure disk space under `asset_cache_dir` and write access to temp dirs. |

---

## Asset types (`asset_type`)

| Type | `asset_dest` | Version check | Installed artifact |
|------|----------------|---------------|---------------------|
| **`binary`** (default) | File path (optional: default `which` or `/usr/local/bin/{{ asset_name }}`) | Runs `{{ _asset_dest }} {{ asset_version_cmd }}`, parses **`asset_version_regex`** | Single executable file |
| **`file`** | Full path to the file | Reads `{{ dirname(asset_dest) }}/.{{ asset_name }}_ansible_version` | One file |
| **`package`** | **Directory** (must exist as parent path; role creates it) | Reads `{{ asset_dest }}/.ansible_version` | Tree of files (archive extract or staged tree) |

---

## Required variables (caller)

| Variable | Description |
|----------|-------------|
| `asset_name` | Logical name (caching, temp prefixes, version sidecar file for `file` type). |
| `asset_type` | `binary` \| `file` \| `package`. |
| `asset_version` | Constraint: plain `1.2.3` (treated as **`>=`**), **`==1.2.3`**, or **`>=1.2.3`**. The role strips the operator and compares with **`ansible.builtin.version`**. |
| `asset_url` | URL or URL **suffix**. Use **`%v`** for the resolved numeric version string. If the value does **not** start with `http://` or `https://`, **`asset_origin_scheme`** (default `https://`) is prepended. Examples: `github.com/org/repo/releases/download/v%v/app-%v-linux-amd64.tar.gz` |

For **`binary`**, you must also pass **`asset_version_cmd`** and **`asset_version_regex`**.

---

## Optional variables

| Variable | Default | Description |
|----------|---------|-------------|
| `asset_dest` | See types above | Final path on the **target** host. |
| `asset_extract` | `true` | If `true`, treat download as archive and extract; if `false`, single blob. |
| `asset_extract_src` | `{{ asset_name }}` | Path inside the archive (after extract) or staging path; **`%v`** is replaced with the resolved version. For **`package`**, a trailing **`/`** means “copy directory **contents**”. |
| `asset_download_locally` | `false` | If `true`: download/cache on controller, optionally repack and `unarchive` on target (air-gap / restricted egress). |
| `asset_cache_enabled` | `true` | Only when `asset_download_locally` is `true`: reuse **`asset_cache_dir`** mirror of remote file. |
| `asset_cache_dir` | `$HOME/.cache/ahmz/server_setup/assets` | Controller cache root (lookup `HOME`). |
| `asset_cache_force_refresh` | `false` | Re-download into cache even if cache file exists. |
| `asset_origin_scheme` | `https://` | Prepended when `asset_url` has no scheme. |
| `asset_use_mirror` | `false` | If `true` and `asset_mirror_base_url` set, fetch from mirror + suffix path. |
| `asset_mirror_base_url` | `""` | Mirror origin (no trailing slash required; role normalizes). |
| `asset_proxy` / `asset_proxy_username` / `asset_proxy_password` | empty | Passed to `get_url` / `unarchive` downloads. With `asset_download_locally`, proxy applies on **localhost**; otherwise on the **target**. |
| `asset_service_stop` | _(unset)_ | If set, **`systemd`** `state: stopped` on the target before install (best-effort). |
| `asset_register_path_var` | _(unset)_ | If set, **`set_fact`** that name to **`_asset_dest`** on the target after install. |

Role defaults live in `defaults/main.yml`; callers override via `include_role` / `import_role` **`vars`**.

---

## Idempotency

- **Install** runs if the destination is **missing** or the installed version does **not** satisfy `asset_version` (`==` or `>=`).
- **Binary**: version from CLI output.
- **File / package**: version from the sidecar file above; missing file ⇒ treated as needing install.

---

## Local staging (`asset_download_locally: true`)

1. Resolve URL, optionally use **cache** on the controller.  
2. Extract or download into a **temp dir** on localhost (or on target if cache off and not staging — see tasks).  
3. For staging: copy selected payload into **`payload/`**, **`community.general.archive`** → `tar.gz`, **`unarchive`** on target under a temp dir, then **`copy`** / **`remote_src`** into `asset_dest`.  
4. **`package`** copies with a **trailing slash** on the source so the **contents** land in `asset_dest`, not an extra nested directory (important for Docker-style tgz layouts).

---

## Examples

### Binary (no extract), e.g. compose-style single file

```yaml
- name: Install docker-compose plugin binary
  ansible.builtin.include_role:
    name: ahmz1833.server_setup.asset
  vars:
    asset_name: docker-compose
    asset_version: "5.1.1"
    asset_url: "github.com/docker/compose/releases/download/v%v/docker-compose-linux-x86_64"
    asset_dest: /usr/local/libexec/docker/cli-plugins/docker-compose
    asset_extract: false
    asset_version_cmd: "version --short"
    asset_version_regex: 'v?([0-9]+\.[0-9]+\.[0-9]+)'
```

### Archive with extract + service stop + path fact

```yaml
- name: Install sing-box
  ansible.builtin.include_role:
    name: ahmz1833.server_setup.asset
  vars:
    asset_name: sing-box
    asset_version: "1.13.3"
    asset_url: "github.com/SagerNet/sing-box/releases/download/v%v/sing-box-%v-linux-amd64.tar.gz"
    asset_dest: /usr/local/bin/sing-box
    asset_extract: true
    asset_extract_src: "sing-box-%v-linux-amd64/sing-box"
    asset_version_cmd: "version"
    asset_version_regex: 'sing-box version ([0-9.]+)'
    asset_service_stop: sing-box
    asset_register_path_var: node_singbox_bin_path
```

### Package directory + controller cache (restricted target)

```yaml
- name: Install static bundle tree
  ansible.builtin.include_role:
    name: ahmz1833.server_setup.asset
  vars:
    asset_name: myapp
    asset_type: package
    asset_version: "2.0.0"
    asset_url: "download.example.com/myapp-%v-linux-amd64.tar.gz"
    asset_dest: /opt/myapp
    asset_extract: true
    asset_extract_src: "myapp-%v/"
    asset_download_locally: true
    asset_proxy: "socks5://127.0.0.1:1080"
```

---

## Galaxy metadata tags

`utility`, `installer`, `downloader` (search hints on Galaxy; this role does **not** define Ansible **task** tags unless you wrap the include).

---

## License

MIT
