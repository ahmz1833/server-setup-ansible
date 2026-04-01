# Ansible role: `ahmz1833.server_setup.nginx`

Production-oriented Nginx reverse proxy and edge stack for **Debian and Ubuntu**: packages, global tuning snippets, YAML-driven virtual hosts, optional ModSecurity (OWASP CRS), real client IP handling, JSON access logs with optional Promtail → Prometheus metrics, rolling configuration backups, and optional `nginx-prometheus-exporter`.

---

## Requirements

| Requirement | Notes |
|-------------|--------|
| **Target OS** | **Debian / Ubuntu** only. Tasks assume `apt`-style packages, `/etc/nginx/sites-available` / `sites-enabled`, and paths used by the stock distro packages. |
| **Ansible** | 2.14+ (`meta/main.yml`). |
| **systemd** | Service and handler management use **`ansible.builtin.systemd`**. |
| **Dependency role** | **`asset`** (`meta/main.yml`) — used to fetch ModSecurity/CRS files, `nginx-prometheus-exporter`, and Promtail binaries when those features run. |
| **Controller** | Trusted-proxy **`url:`** entries are fetched with **`ansible.builtin.uri`** from the **control node** (`delegate_to: localhost`, `run_once: true`). The control host needs outbound access to those URLs at playbook time. |

### Architecture note (metrics / Promtail)

`tasks/metrics.yml` and `tasks/promtail.yml` install **linux amd64** release artifacts. For **ARM** or other arches, override the install path (e.g. fork those tasks or install packages yourself) before relying on this role unchanged.

---

## Enabling the role

Nothing runs until you set:

```yaml
nginx_managed: true
```

Optional **playbook-level** variables read in `defaults/main.yml`:

| Playbook / extra var | Used as |
|----------------------|---------|
| `enable_ipv6` | `nginx_enable_ipv6` — adds `listen [::]:…` when `true`. |
| `is_iran` | `nginx_download_locally` — passed into **`asset`** for restricted egress (same pattern as other roles). |

---

## Ansible tags

Applied in `tasks/main.yml`:

| Tag | Scope |
|-----|--------|
| `nginx` | Entire role (via sub-tags on each import). |
| `nginx-install` | Packages (`install.yml`). |
| `nginx-waf` | WAF install + ModSecurity config (`waf.yml`). |
| `nginx-config` | Main nginx tuning + real IP (`config.yml`, `realip.yml`). |
| `nginx-realip` | Real-IP / trusted-proxy config only (`realip.yml`). |
| `nginx-sites` | Site templates and `sites-enabled` links (`sites.yml`). |
| `nginx-monitoring` | Exporter + Promtail (`metrics.yml`, `promtail.yml`). |
| `nginx-promtail` | Promtail only. |
| `nginx-error-pages` | Static error HTML (`error_pages.yml`; site task file is also tagged for ordering). |

---

## Global variables (summary)

Defaults live in `defaults/main.yml`. Highlights:

| Area | Variables (examples) |
|------|----------------------|
| **Control** | `nginx_managed`, `nginx_remove_default_site`, `nginx_web_user` / `nginx_web_group` |
| **Backup** | `nginx_backup_enabled`, `nginx_backup_dir` — backs up `nginx.conf` and, before site updates, `sites-available/*.conf` under `{{ nginx_backup_dir }}/sites/`. Keeps the **10** newest `nginx.conf` backups and **5** newest backups **per site** prefix. |
| **Tuning** | `nginx_worker_connections`, gzip (`nginx_enable_gzip`, …), rate limit zone (`nginx_enable_rate_limit`, `nginx_rate_limit_*`) |
| **Real IP** | `nginx_real_ip_header`, `nginx_trusted_proxies`, `nginx_trusted_proxies_extra`, `nginx_ipset_data_dir` |
| **Security** | `nginx_server_tokens`, `nginx_hide_server_header` (headers-more), `nginx_proxy_intercept_errors`, default `header_*` overrides |
| **SSL defaults** | `nginx_ssl_*`, `nginx_acme_challenge_enabled`, `nginx_default_acme_dir` |
| **WAF** | `nginx_waf_install`, `nginx_waf_global_settings` |
| **Exporter** | `nginx_exporter_install`, `nginx_exporter_enabled`, `nginx_exporter_version`, `nginx_exporter_listen_address`, `nginx_dummy_metric_port` |
| **Promtail** | `nginx_promtail_install`, `nginx_promtail_enabled`, `nginx_promtail_version`, `nginx_promtail_port`, `nginx_promtail_loki_url`, `nginx_promtail_extra_labels` |
| **Errors** | `nginx_standard_error_codes` — drives built-in global error pages and `custom_errors` matching |

---

## Site definitions: `nginx_sites`

`nginx_sites` is a **list of dictionaries**; each item is one `server_name` group. The Jinja template (`templates/site.conf.j2`) emits HTTP → HTTPS redirect server blocks when `ssl_enabled` is true, applies TLS, optional ModSecurity, proxy settings, locations, and JSON `json_analytics` access logs (with `$endpoint_name` per location for cardinality-safe metrics).

### Site keys (reference)

| Key | Description |
|-----|-------------|
| `domain` | **Required.** Primary `server_name`. |
| `aliases` | Extra server names. |
| `enabled` | Symlink in `sites-enabled` (default `true`). |
| `ssl_enabled` | HTTPS on 443 (default `false`). |
| `ssl_cert` / `ssl_key` | Paths; default under `nginx_ssl_cert_dir` / `domain`. |
| `acme_challenge_enabled` | Per-site ACME webroot (also global `nginx_acme_challenge_enabled`). |
| `upstream` | `proxy_pass` for `/` when not a pure redirect/static `root`. |
| `redirect_to` | `return 301` target; original URI is appended (`$request_uri`). |
| `waf_enabled` | ModSecurity for this vhost. |
| `waf_config` / `waf_extra_rules` | Inline rules vs default `modsec/main.conf` include. |
| `header_*` | Override global security headers (e.g. `header_csp`, `header_hsts`). |
| `cache_control` | Site-level `Cache-Control` when using the header macro. |
| `nginx_config` | Dict of `directive: value` → `directive value;` in `server`. |
| `extra_config` | Raw nginx fragment in `server`. |
| `locations` | List of location dicts: `path`, `upstream`, `rate_limit`, `log_prefix`, `cache_control`, `nginx_config`, `extra_config`, `proxy_intercept_errors`, … |
| `custom_errors` | List of `{ match: '^404$' \| '^5..$', template: 'path/to/template.j2' }` — regex matches keys of `nginx_standard_error_codes`. Templates receive `error_code` and `error_message`. Set `proxy_intercept_errors: "on"` where you need upstream errors mapped to those pages. |

**Declarative cleanup:** Any **symlink basename** under `/etc/nginx/sites-enabled` that is **not** listed in `nginx_sites` (as `domain.conf`) and is **not** the distro `default` (when `nginx_remove_default_site` is false) is **removed**. An **empty** `nginx_sites` therefore removes **all** such symlinks except an intentionally kept `default`. Treat `nginx_sites` as the full desired set for managed hosts.

Full worked examples: **`examples/sites.yml`** (reference only; not loaded automatically).

---

## Trusted proxies (real IP)

`nginx_trusted_proxies` (plus `nginx_trusted_proxies_extra`) may contain:

- **Plain strings:** CIDRs, e.g. `10.0.0.0/8`.
- **`{ ipset: "name" }`:** Reads `{{ nginx_ipset_data_dir }}/name.save` and parses `add …` lines into CIDRs.
- **`{ url: "https://…" }`:** Fetched **once** from the **controller**; body lines parsed as IPv4/IPv6 CIDRs (comments `#` skipped).

Example:

```yaml
nginx_real_ip_header: "X-Forwarded-For"
nginx_trusted_proxies:
  - "10.0.0.0/8"
  - { ipset: "cloudflare-v4" }
  - { url: "https://www.cloudflare.com/ips-v4" }
```

---

## WAF (ModSecurity + CRS)

When `nginx_waf_install` is true, the role installs distro packages (`libnginx-mod-http-modsecurity`, `modsecurity-crs`), drops recommended configs via **`asset`**, merges `nginx_waf_global_settings` into `/etc/modsecurity/modsecurity.conf`, and renders `/etc/nginx/modsec/main.conf`. Per-site behavior is controlled with `waf_enabled` and optional `waf_config` / `waf_extra_rules` in `nginx_sites`.

**How the template chooses ModSecurity mode**

- **`waf_enabled: true`** with **no** `waf_config` and **no** `waf_extra_rules` → `modsecurity_rules_file /etc/nginx/modsec/main.conf;` (shared CRS chain from `main.conf`).
- **`waf_enabled: true`** and **either** `waf_config` **or** `waf_extra_rules` is set → an **inline** `modsecurity_rules '…'` block is generated: it includes `modsecurity.conf`, your `waf_config` key/value pairs, `crs-setup.conf`, all CRS `rules/*.conf`, then appends `waf_extra_rules` (use this for `SecRuleRemoveById`, path-specific `ctl:`, or custom `SecRule` lines).

---

## Examples

### WAF — HTTPS API using the default CRS chain (`main.conf`)

Minimal site: TLS, reverse proxy, ModSecurity on, no per-site overrides. OWASP CRS is loaded as defined in `/etc/nginx/modsec/main.conf`.

```yaml
nginx_managed: true
nginx_waf_install: true

nginx_sites:
  - domain: api.example.com
    ssl_enabled: true
    upstream: "http://127.0.0.1:8080"
    waf_enabled: true
```

### WAF — Larger uploads and relaxed body limits (inline `waf_config`)

Use `waf_config` when you need **per-vhost** ModSecurity directives (here: allow bigger request bodies). This switches that vhost to the **inline** rules block (see above).

```yaml
nginx_sites:
  - domain: uploads.example.com
    ssl_enabled: true
    upstream: "http://127.0.0.1:9000"
    waf_enabled: true
    waf_config:
      SecRuleEngine: "On"
      SecRequestBodyLimit: "52428800"   # 50 MiB
      SecRequestBodyNoFilesLimit: "524288"
```

### WAF — False positives, webhooks, and custom rules (`waf_extra_rules`)

Append raw ModSecurity lines after the CRS includes: whitelist IPs, remove noisy rule IDs on specific paths, or add custom blocks. Combine with `waf_config` if you need both.

```yaml
nginx_sites:
  - domain: secure.example.com
    ssl_enabled: true
    upstream: "http://127.0.0.1:9000"
    waf_enabled: true
    waf_config:
      SecRuleEngine: "On"
    waf_extra_rules: |
      SecRule REMOTE_ADDR "@ipMatch 10.0.0.0/8" \
        "id:1000,phase:1,allow,nolog"

      SecRule REQUEST_URI "@beginsWith /api/webhook" \
        "id:1001,phase:1,ctl:ruleRemoveById=942100"

      SecRule REQUEST_HEADERS:User-Agent "@contains BadBot" \
        "id:1002,phase:1,deny,status:403,msg:'Bad bot blocked'"
```

### WAF — Sensitive admin path with stricter rate limiting

Reuse the global limit zone from `defaults/main.yml` (`nginx_rate_limit_zone_name` / `nginx_rate_limit_rate`) and tighten burst on `/admin` while keeping the WAF on the same vhost.

```yaml
nginx_sites:
  - domain: app.example.com
    ssl_enabled: true
    upstream: "http://127.0.0.1:3000"
    waf_enabled: true
    locations:
      - path: /admin
        upstream: "http://127.0.0.1:3000"
        rate_limit:
          zone: global_limit
          burst: 5
          nodelay: true
        log_prefix: admin
```

### Global ModSecurity tuning (`nginx_waf_global_settings`)

These directives are written into **`/etc/modsecurity/modsecurity.conf`** for **all** WAF-enabled sites. Defaults are in `defaults/main.yml`; override in group_vars when you want a single policy everywhere.

```yaml
nginx_managed: true
nginx_waf_install: true

nginx_waf_global_settings:
  SecRuleEngine: "On"
  SecRequestBodyAccess: "On"
  SecRequestBodyLimit: "13107200"
  SecRequestBodyNoFilesLimit: "131072"
  SecResponseBodyAccess: "On"
  SecResponseBodyLimit: "524288"
```

### Skip WAF packages entirely

Useful for dev mirrors, internal terminators, or when ModSecurity is provided elsewhere.

```yaml
nginx_managed: true
nginx_waf_install: false

nginx_sites:
  - domain: internal.dev.local
    ssl_enabled: false
    upstream: "http://127.0.0.1:4000"
```

### ACME HTTP-01 on the edge + HTTPS reverse proxy (with WAF)

Global webroot for challenges (see `nginx_default_acme_dir`); one public site with TLS and CRS.

```yaml
nginx_managed: true
nginx_waf_install: true
nginx_acme_challenge_enabled: true
nginx_default_acme_dir: /var/www/acme

nginx_sites:
  - domain: www.example.com
    ssl_enabled: true
    acme_challenge_enabled: true
    upstream: "http://127.0.0.1:8080"
    waf_enabled: true
```

### Static site — `root` and `try_files` (no `upstream`)

Omit **`upstream`** so the template does not add a default `location /` proxy. Set **`root`** / **`index`** at the server level and a **`locations`** entry for `/` with **`try_files`** for typical SPA or plain static trees.

The role only installs Nginx configuration; **create the document root and deploy files** yourself (e.g. `copy`, `synchronize`, or your release pipeline).

```yaml
nginx_managed: true

nginx_sites:
  - domain: static.example.com
    ssl_enabled: true
    nginx_config:
      root: /var/www/static.example.com
      index: index.html
    locations:
      - path: /
        nginx_config:
          try_files: "$uri $uri/ =404"
```

HTTP-only internal example (same idea, no TLS):

```yaml
nginx_sites:
  - domain: assets.internal.lan
    ssl_enabled: false
    nginx_config:
      root: /var/www/assets
      index: index.html
    locations:
      - path: /
        nginx_config:
          try_files: "$uri $uri/ =404"
```

### Mixed fleet: one WAF site and one plain internal vhost

Illustrates that only sites with `waf_enabled: true` run ModSecurity; others stay a normal proxy.

```yaml
nginx_sites:
  - domain: public.example.com
    ssl_enabled: true
    upstream: "http://127.0.0.1:3000"
    waf_enabled: true

  - domain: internal.lan
    ssl_enabled: false
    upstream: "http://127.0.0.1:5000"
    header_x_frame_options: ""
    header_hsts: ""
```

More patterns (headers, redirects, static sites, raw `extra_config`) live in **`examples/sites.yml`**.

---

## Observability

- **JSON logs:** `logging.conf.j2` defines `json_analytics`; site template uses it for access logs. Location blocks set `$endpoint_name` to the location path so Promtail can aggregate without using raw `$request_uri` labels.
- **Promtail:** When `nginx_promtail_install` is true, a config and systemd unit are installed; start is gated by `nginx_promtail_enabled`.
- **Exporter:** `stub_status` on localhost + `nginx-prometheus-exporter`; enable with `nginx_exporter_enabled`.

---

## Example playbook snippet

```yaml
- hosts: edge
  roles:
    - role: ahmz1833.server_setup.nginx
      vars:
        nginx_managed: true
        nginx_waf_install: true
        nginx_exporter_enabled: true
        nginx_sites:
          - domain: app.example.com
            ssl_enabled: true
            upstream: "http://127.0.0.1:3000"
            nginx_config:
              client_max_body_size: "100M"
```

---

## Handlers

- **`Reload nginx`:** `nginx -t` then `systemctl reload nginx`.
- **`Restart nginx`:** `nginx -t` then `systemctl restart nginx`.
- **`Restart nginx_exporter` / `Restart promtail`:** Restart the respective unit; exporter handler runs only when `nginx_exporter_enabled` is true (Promtail handler when `nginx_promtail_enabled` is true).

---

## License

MIT
